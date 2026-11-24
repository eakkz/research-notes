# Airflow pointers for Windows Authentication to SQL Server

Collected 2026-11-23. Versions checked: Airflow 3.4.0 (https://pypi.org/project/apache-airflow/3.4.0/), Helm chart 1.22.0, the latest chart release, dated 2026-06-01 (https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/RELEASE_NOTES.rst), apache-airflow-providers-microsoft-mssql 4.7.3, apache-airflow-providers-odbc 4.13.1 and pymssql 2.4.6. Anything the sources do not state is marked "not confirmed".

## 1. Helm chart Kerberos values

`kerberos.enabled` (default false) turns on three things: the chart renders `kerberos.config` as `krb5.conf` into the Airflow ConfigMap (https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/templates/configmaps/configmap.yaml#L80-L83), it adds a `kerberos-keytab` Secret volume and a `kerberos-ccache` emptyDir volume to worker pods, and it mounts the keytab, `krb5.conf` and ccache read-only into the task container (https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/files/pod-template-file.kubernetes-helm-yaml#L151-L163 and #L284-L290 in the same file).

The keytab comes from a Secret named `<release>-kerberos-keytab` with key `kerberos.keytab`. If you set `kerberos.keytabBase64Content`, the chart creates that Secret itself. The chart only does this when `kerberos.enabled` is also true (https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/values.yaml#L624-L650, https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/templates/secrets/kerberos-keytab-secret.yaml).

Schema descriptions (https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/values.schema.json):
- `ccacheMountPath` (default `/var/kerberos-ccache`) is the "path to mount shared volume for kerberos credentials cache". Together with `ccacheFileName` (default `cache`), it sets `[kerberos] ccache` in the generated airflow.cfg (https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/values.yaml#L4277-L4281).
- `configPath` (default `/etc/krb5.conf`) is where `krb5.conf` is mounted.
- `keytabPath`, `principal` and `reinitFrequency` feed `[kerberos] keytab`, `principal` and `reinit_frequency`.
- When the sidecar or init container is enabled, the task container also gets `KRB5_CONFIG` and `KRB5CCNAME` pointing at those paths (pod template #L113-L119).

The sidecar and init container work like this:
- The sidecar is a container named `worker-kerberos` that runs `airflow kerberos` (the continuous renewer). It is the only container that mounts the keytab and a writable ccache.
- The init container, `kerberos-init`, runs `airflow kerberos -o` once before the task container starts (pod template #L59-L66 and #L165-L191).
- In 1.22.0 the preferred keys are `workers.celery.kerberosSidecar` / `workers.kubernetes.kerberosSidecar` and the matching `kerberosInitContainer` keys.
- The older top-level `workers.kerberosSidecar` and `workers.kerberosInitContainer` are deprecated but still read, combined with OR logic (values.yaml #L937-L1020, https://github.com/apache/airflow/pull/61881).

When each piece was added, from the chart release notes (RELEASE_NOTES.rst link above):
- The `kerberos` block (`enabled`, `ccacheMountPath`, `configPath`) and `workers.kerberosSidecar` were already in chart 1.0.0 (https://github.com/apache/airflow/blob/helm-chart/1.0.0/chart/values.yaml#L288-L292).
- `keytabBase64Content`: 1.4.0 (https://github.com/apache/airflow/pull/19054).
- Init container plus sidecar model (`workers.kerberosInitContainer`): 1.12.0 (https://github.com/apache/airflow/pull/35548).
- Sidecar and init container in the KubernetesExecutor pod template: 1.14.0 (https://github.com/apache/airflow/pull/38815).
- `workers.celery|kubernetes.kerberosInitContainer`: 1.19.0 (https://github.com/apache/airflow/pull/60751).
- `workers.celery|kubernetes.kerberosSidecar`, and deprecation of the top-level keys: 1.20.0 (https://github.com/apache/airflow/pull/61881).

Unreleased changes on main, where Chart.yaml says 2.0.0 (https://github.com/apache/airflow/blob/main/chart/Chart.yaml):
- The sidecar moves to a native sidecar (`initContainers` with `restartPolicy: Always`) with a `klist -s` startup probe.
- The `kerberosInitContainer` values are removed (https://github.com/apache/airflow/blob/main/chart/newsfragments/72555.significant.rst).
- A new `securityContexts.disableDefaults` option lets OpenShift SCC assign UIDs instead of the chart hard-coding them (https://github.com/apache/airflow/blob/main/chart/newsfragments/70238.significant.rst).
- Which chart release will ship these: not confirmed.

The chart docs have no dedicated Kerberos page at 1.22.0. The production guide only lists the `<RELEASE_NAME>-kerberos-keytab` Secret name (https://airflow.apache.org/docs/helm-chart/stable/production-guide.html).

## 2. The `airflow kerberos` renewer and the `[kerberos]` section

The CLI reference describes `airflow kerberos` as "Start a kerberos ticket renewer". It takes `[principal]`, `-k/--keytab` and `-o/--one-time` (https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#kerberos). In standard mode it "will run endlessly". In one-time mode it runs once and exits, and "in case of failure the main task won't spin up" (https://airflow.apache.org/docs/apache-airflow/stable/security/kerberos.html).

The source (https://github.com/apache/airflow/blob/3.4.0/airflow-core/src/airflow/security/kerberos.py#L67-L145) builds a command of the form `kinit -f|-F -a|-A [-r <reinit_frequency>m] -k -t <keytab> -c <ccache> <principal>`. In standard mode it then sleeps `reinit_frequency` seconds and repeats (same file #L197-L217). It needs a `kinit` binary. The official image's runtime packages include `krb5-user`, `freetds-bin`, `tdsodbc` and `unixodbc`, and `INSTALL_MSSQL_CLIENT` (default true) installs `msodbcsql18` (https://github.com/apache/airflow/blob/3.4.0/Dockerfile, https://airflow.apache.org/docs/docker-stack/build-arg-ref.html).

`[kerberos]` keys (https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#kerberos):
- `ccache`: default `/tmp/airflow_krb5_ccache`.
- `keytab`: default `airflow.keytab`.
- `principal`: default `airflow`.
- `reinit_frequency`: default 3600.
- Also `kinit_path`, `forwardable`, `include_ip`, and `include_renewal_lifetime` (added in 3.4.0).

The Kerberos page warns that the default ccache sits in a world-readable directory. It recommends a directory with mode 0700 owned by the Airflow user (https://airflow.apache.org/docs/apache-airflow/stable/security/kerberos.html).

Units for `reinit_frequency` are inconsistent between sources. The source comment says seconds, and the code sleeps that many seconds. The chart schema describes `reinitFrequency` as "how often (in minutes)" (values.schema.json link above). Treat it as seconds, per the code.

The deployment guide recommends that only the renewer container sees the keytab, and that workers read the refreshed ticket from a shared volume. It states that "this concept is implemented in the Helm Chart" (https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/production-deployment.html#production-deployment-kerberos). The Kerberos page notes that "not all hooks have been adjusted" and only documents the Hive hook. MsSqlHook and OdbcHook are not mentioned (https://airflow.apache.org/docs/apache-airflow/stable/security/kerberos.html).

## 3. KubernetesExecutor and KubernetesPodOperator

KubernetesExecutor: yes, per the chart. The schema describes `workers.kubernetes.kerberosSidecar` as "Kerberos sidecar for pods created with pod-template-file" (values.schema.json link above). The chart's pod template adds `worker-kerberos` and `kerberos-init` and sets `KRB5CCNAME` on the `base` container (pod template link above). The executor builds worker pods from `[kubernetes_executor] pod_template_file`, which allows sidecar containers after `base` (https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html).

In chart 1.22.0 the sidecar is still an ordinary container in `spec.containers`. The PR that moves it to a native sidecar reports that, before the change, "the task exited successfully, but its Pod stayed `Running` for 24m37s and required manual deletion" (https://github.com/apache/airflow/pull/72555, merged to main 2026-11-03, unreleased). An earlier attempt at the same problem, tested on OpenShift, was closed without merging (https://github.com/apache/airflow/pull/35748). The native sidecar needs the `SidecarContainers` feature gate on Kubernetes 1.30 to 1.32 (72555 newsfragment above).

KubernetesPodOperator: the docs say KPO defines its pod from operator arguments, `full_pod_spec`, `pod_template_file` or `pod_template_dict`. They say nothing about Kerberos or the chart's sidecar (https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/operators.html). Whether the chart sidecar reaches KPO pods: not confirmed. The docs are silent, and the chart ties the sidecar only to the executor pod template.

## 4. MsSqlHook and OdbcHook

MsSqlHook 4.7.3 uses pymssql (`pymssql>=2.3.5`). `get_conn` calls `pymssql.connect(server=host, user=login, password=password, database=schema, port=port, **extras)`, passing every extra except `sqlalchemy_scheme` (https://pypi.org/project/apache-airflow-providers-microsoft-mssql/4.7.3/, https://github.com/apache/airflow/blob/providers-microsoft-mssql/4.7.3/providers/microsoft/mssql/src/airflow/providers/microsoft/mssql/hooks/mssql.py#L101-L112). The connection page lists Login and Password as required and never mentions Windows Authentication, domain logins or Kerberos (https://airflow.apache.org/docs/apache-airflow-providers-microsoft-mssql/stable/connections/mssql.html).

What that means for Windows auth:
- NTLM: a `DOMAIN\user` login would reach FreeTDS's domain-login path (section 5). This is inferred from the code, not confirmed in Airflow docs.
- Kerberos with an empty login: not confirmed.
- On main only, a `dbapi_driver` extra set to `mssql_python` switches to Microsoft's mssql-python driver (https://github.com/apache/airflow/blob/main/providers/microsoft/mssql/src/airflow/providers/microsoft/mssql/hooks/mssql.py). It is not in the 4.7.3 release.

OdbcHook 4.13.1 uses pyodbc (`pyodbc>=5.0.0`) (https://pypi.org/project/apache-airflow-providers-odbc/4.13.1/). It builds the connection string as follows (https://github.com/apache/airflow/blob/providers-odbc/4.13.1/providers/odbc/src/airflow/providers/odbc/hooks/odbc.py#L126-L159):
- `DRIVER`, `DSN`, `SERVER` and `DATABASE`, from the connection fields.
- `UID` only if login is set, and `PWD` only if password is set.
- `PORT`.
- Every other extra appended verbatim as `key=value;`. Excluded: `driver`, `dsn`, `connect_kwargs`, `sqlalchemy_scheme` and `placeholder`.

So for a trusted connection, leave login and password empty and put the driver's keyword in extras. The Airflow example spells it `"TrustedConnection": "Yes"`. Because the hook passes keys unchanged, the driver's own keyword spelling decides; the page links Microsoft's keyword reference for that (https://airflow.apache.org/docs/apache-airflow-providers-odbc/stable/connections/odbc.html).

`connect_kwargs` is passed to `pyodbc.connect` (same page). The `driver` extra is ignored unless `[providers.odbc] allow_driver_in_extra = True` (`AIRFLOW__PROVIDERS_ODBC__ALLOW_DRIVER_IN_EXTRA=true`, default False). Other ways to set it are the hook's `driver` argument, `hook_params={"driver": ...}` on SQL operators, or patching `OdbcHook.default_driver` in `local_settings.py` (odbc.py #L32-L41 and #L97-L114, https://github.com/apache/airflow/blob/providers-odbc/4.13.1/providers/odbc/provider.yaml).

## 5. pymssql and FreeTDS Kerberos and NTLM

pymssql 2.4.6 was released 2026-10-27 (https://pypi.org/project/pymssql/2.4.6/). Its manylinux and musllinux wheels are built with `dev/build.py --enable-krb5 --static-freetds`. The macOS build does not pass `--enable-krb5` (https://github.com/pymssql/pymssql/blob/v2.4.6/pyproject.toml#L69-L92). The changelog records "Enable krb5 in Linux wheels" in 2.2.3, "this time for real" in 2.2.5, and an explicit krb5 link in 2.2.6 (https://github.com/pymssql/pymssql/blob/master/ChangeLog.rst).

I opened the cp312 manylinux x86_64 wheel (https://files.pythonhosted.org/packages/42/f9/5c561c35a0be1e234f16658b9a76e5adacbe0a77c91d93390ba5ab3e5ca5/pymssql-2.4.6-cp312-cp312-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl):
- `pymssql.libs/` bundles `libgssapi_krb5`, `libkrb5`, `libk5crypto`, `libkrb5support`, `libcom_err`, and OpenSSL `libssl`/`libcrypto` 1.1.1k.
- `_mssql` contains `freetds v1.5.19` and `gss_init_sec_context` symbols.
- The Linux wheels therefore include GSSAPI/Kerberos.

Which call pattern triggers Kerberos (for example `user=None`) is not documented: the `connect()` reference only documents user and password (https://github.com/pymssql/pymssql/blob/master/docs/ref/pymssql.rst). Not confirmed.

FreeTDS does support NTLM domain logins over TCP/IP with TDS 7.0 or later. Without SSPI or Kerberos, use `DOMAIN\username` and the domain password; "when FreeTDS sees the '\' character, it automatically chooses a domain login" (https://www.freetds.org/userguide/domains.html). In `freetds.conf`, `use ntlmv2` defaults to yes and `use lanman` to no (https://www.freetds.org/userguide/freetdsconf.html).

On non-Windows hosts, FreeTDS Kerberos gives single sign-on. It needs a keytab from Active Directory and `kinit` (https://www.freetds.org/userguide/kerberos.html). The related `freetds.conf` keys are `realm`, `SPN` (default `MSSQLSvc/fqdn:port`), `enable gssapi delegation` and `mutual authentication` (freetdsconf.html above).

## Reading list (opened and confirmed)

1. https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/production-deployment.html#production-deployment-kerberos
2. https://airflow.apache.org/docs/apache-airflow/stable/security/kerberos.html
3. https://github.com/apache/airflow/blob/helm-chart/1.22.0/chart/files/pod-template-file.kubernetes-helm-yaml
4. https://github.com/apache/airflow/pull/72555
5. https://www.freetds.org/userguide/domains.html
