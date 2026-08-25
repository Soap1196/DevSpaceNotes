# Stock RHEL 10 OpenShift Dev Spaces with Nexus, Conjur, and Db2

This runbook configures a stock RHEL 10 container through `devfile.yaml`. It does not build or publish a custom image: Dev Spaces lifecycle commands install user-space tools and configuration from approved Nexus repositories.

The workspace uses enterprise TLS trust, Python with `uv`, Java, Maven, the IBM Db2 CLI driver, and CyberArk Conjur Kubernetes authentication. Replace every `<...>` placeholder before use. Do not commit passwords, Conjur tokens, private keys, or kubeconfig files.

## What the devfile can and cannot configure

Dev Spaces normally runs workspace containers with an arbitrary non-root OpenShift UID. Therefore a lifecycle command cannot run `dnf`, modify `/etc/pki`, or change the image's Java installation. The devfile below handles this without elevated permissions by:

- downloading approved tool archives from Nexus into `$HOME/.local/share/devspace-tools`;
- using `SSL_CERT_FILE` and `CURL_CA_BUNDLE` to trust the corporate PEM bundle without editing `/etc`;
- creating a user-owned Java truststore; and
- setting all Java, Maven, uv, and Db2 paths in the container environment.

Three inputs must be provided outside the devfile because they are cluster or security-administration concerns:

1. The stock RHEL 10 image must provide `/bin/sh`, `curl`, `tar`, `gzip`, and CA bundle support. These are the bootstrap tools; Dev Spaces cannot install them when running non-root.
2. An OpenShift administrator must bind the workspace pod to the approved service account. A Devfile container component does not grant service-account or RBAC privileges.
3. The CyberArk administrator must configure the Conjur Kubernetes authenticator and load its policy. The devfile only consumes that configured integration.

## Prerequisites

Obtain these values from the responsible platform teams.

| Item | Example |
| --- | --- |
| Stock RHEL 10 image | `registry.redhat.io/rhel10/rhel:10.0` |
| Dev Spaces dashboard | `https://devspaces.apps.example.com` |
| Git repository directory under `/projects` | `<repository-directory>` |
| Corporate CA bundle in the Git repository | `certificates/company-ca.pem` |
| Nexus URL | `https://nexus.example.com` |
| Conjur appliance and account | `https://conjur.example.com`, `<conjur-account>` |
| Conjur Kubernetes authenticator | `authn-k8s/openshift` |
| Db2 endpoint | `db2.internal.example.com:50001/SALES` |

Create raw repositories or hosted artifacts in Nexus for the following pinned downloads. The URLs in the devfile are examples; use the artifact names and checksums approved by your organization.

| Artifact | Example Nexus location |
| --- | --- |
| uv release archive | `/repository/devspace-tools/uv/0.8.15/uv-x86_64-unknown-linux-gnu.tar.gz` |
| CPython standalone archive used by uv | `/repository/python-build-standalone` |
| JDK archives | `/repository/devspace-tools/jdk/{8,11,17,21}/jdk-<version>-linux-x64.tar.gz` |
| Maven archive | `/repository/devspace-tools/maven/3.9.9/apache-maven-3.9.9-bin.tar.gz` |
| IBM Db2 CLI driver | `/repository/db2-drivers/12.1.0/linuxx64_odbc_cli.tar.gz` |
| Python package index | `/repository/pypi-internal/simple` |
| Maven repository group | `/repository/maven-public/` |

The workspace namespace needs DNS and egress access to Nexus and Conjur on TCP 443 and to Db2 on its TLS port, usually TCP 50001. When the namespace uses default-deny `NetworkPolicy` resources, allow only those destinations. The Db2 team must also allow traffic from the OpenShift cluster's egress addresses or network range.

## Add the corporate CA bundle

Place the public enterprise root and intermediate CA certificates in PEM format at this repository path:

```text
certificates/company-ca.pem
```

This file contains public trust material, not a secret. Its contents must validate the certificates presented by Nexus, Conjur, and Db2. The devfile references it at `/projects/<repository-directory>/certificates/company-ca.pem`; replace `<repository-directory>` with the directory name Dev Spaces uses for this Git repository.

The devfile does not bypass certificate checks. In particular, do not use `curl -k`, `UV_INSECURE_HOST`, or Db2 options that disable TLS validation.

## Add `devfile.yaml`

Place the following `devfile.yaml` in the root of the Git repository. The `postStart` command is idempotent: it downloads an archive only when its installed tool is absent, then recreates the user-local Maven, Db2, and Java trust configuration.

```yaml
schemaVersion: 2.2.2
metadata:
  name: rhel10-nexus-conjur-db2
attributes:
  controller.devfile.io/scc: restricted
components:
  - name: rhel10
    container:
      image: registry.redhat.io/rhel10/rhel:10.0
      mountSources: true
      sourceMapping: /projects
      memoryLimit: 4Gi
      cpuLimit: "2"
      env:
        - name: HOME
          value: /tmp/devspace-home
        - name: DEVSPACE_TOOLS
          value: /tmp/devspace-home/.local/share/devspace-tools
        - name: CORPORATE_CA
          value: /projects/<repository-directory>/certificates/company-ca.pem
        - name: SSL_CERT_FILE
          value: /projects/<repository-directory>/certificates/company-ca.pem
        - name: CURL_CA_BUNDLE
          value: /projects/<repository-directory>/certificates/company-ca.pem
        - name: JAVA8
          value: /tmp/devspace-home/.local/share/devspace-tools/jdk-8
        - name: JAVA11
          value: /tmp/devspace-home/.local/share/devspace-tools/jdk-11
        - name: JAVA17
          value: /tmp/devspace-home/.local/share/devspace-tools/jdk-17
        - name: JAVA21
          value: /tmp/devspace-home/.local/share/devspace-tools/jdk-21
        - name: JAVA_HOME
          value: /tmp/devspace-home/.local/share/devspace-tools/jdk-21
        - name: MAVEN_HOME
          value: /tmp/devspace-home/.local/share/devspace-tools/maven
        - name: DB2_HOME
          value: /tmp/devspace-home/.local/share/devspace-tools/db2/clidriver
        - name: DB2DSDRIVER_CFG_PATH
          value: /tmp/devspace-home/.config/db2
        - name: PATH
          value: /tmp/devspace-home/.local/share/devspace-tools/uv:/tmp/devspace-home/.local/share/devspace-tools/maven/bin:/tmp/devspace-home/.local/share/devspace-tools/jdk-21/bin:/tmp/devspace-home/.local/share/devspace-tools/db2/clidriver/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        - name: LD_LIBRARY_PATH
          value: /tmp/devspace-home/.local/share/devspace-tools/db2/clidriver/lib
        - name: JAVA_TOOL_OPTIONS
          value: -Djavax.net.ssl.trustStore=/tmp/devspace-home/.local/share/devspace-tools/company-truststore.p12 -Djavax.net.ssl.trustStoreType=PKCS12
        - name: UV_INDEX_URL
          value: https://nexus.example.com/repository/pypi-internal/simple
        - name: UV_NATIVE_TLS
          value: "true"
        - name: UV_PYTHON_INSTALL_MIRROR
          value: https://nexus.example.com/repository/python-build-standalone
        - name: CONJUR_APPLIANCE_URL
          value: https://conjur.example.com
        - name: CONJUR_ACCOUNT
          value: <conjur-account>
        - name: CONJUR_AUTHN_URL
          value: https://conjur.example.com/authn-k8s/openshift
        - name: CONJUR_AUTHN_LOGIN
          value: host/devspaces/devspace-db2
commands:
  - id: bootstrap-tools
    exec:
      component: rhel10
      workingDir: /projects/<repository-directory>
      commandLine: |
        sh -ec '
          test -r "$CORPORATE_CA"
          mkdir -p "$HOME/.local/bin" "$HOME/.m2" "$DB2DSDRIVER_CFG_PATH" "$DEVSPACE_TOOLS"

          fetch() {
            curl --fail --location --show-error --cacert "$CORPORATE_CA" --output "$2" "$1"
          }

          if ! test -x "$DEVSPACE_TOOLS/uv/uv"; then
            mkdir -p "$DEVSPACE_TOOLS/uv"
            fetch "https://nexus.example.com/repository/devspace-tools/uv/0.8.15/uv-x86_64-unknown-linux-gnu.tar.gz" /tmp/uv.tar.gz
            tar -xzf /tmp/uv.tar.gz -C /tmp
            install -m 0755 /tmp/uv-x86_64-unknown-linux-gnu/uv "$DEVSPACE_TOOLS/uv/uv"
            rm -rf /tmp/uv.tar.gz /tmp/uv-x86_64-unknown-linux-gnu
          fi

          if ! uv python find 3.12 >/dev/null 2>&1; then
            uv python install 3.12
          fi

          for java_version in 8 11 17 21; do
            eval "java_home=\${JAVA$java_version}"
            if ! test -x "$java_home/bin/java"; then
              rm -rf "$java_home" /tmp/jdk
              fetch "https://nexus.example.com/repository/devspace-tools/jdk/$java_version/jdk-$java_version-linux-x64.tar.gz" /tmp/jdk.tar.gz
              mkdir -p /tmp/jdk
              tar -xzf /tmp/jdk.tar.gz -C /tmp/jdk --strip-components=1
              mv /tmp/jdk "$java_home"
              rm -f /tmp/jdk.tar.gz
            fi
          done

          if ! test -x "$MAVEN_HOME/bin/mvn"; then
            rm -rf "$MAVEN_HOME" /tmp/maven
            fetch "https://nexus.example.com/repository/devspace-tools/maven/3.9.9/apache-maven-3.9.9-bin.tar.gz" /tmp/maven.tar.gz
            mkdir -p /tmp/maven
            tar -xzf /tmp/maven.tar.gz -C /tmp/maven --strip-components=1
            mv /tmp/maven "$MAVEN_HOME"
            rm -f /tmp/maven.tar.gz
          fi

          if ! test -x "$DB2_HOME/bin/db2cli"; then
            rm -rf "$DEVSPACE_TOOLS/db2" /tmp/db2
            fetch "https://nexus.example.com/repository/db2-drivers/12.1.0/linuxx64_odbc_cli.tar.gz" /tmp/db2.tar.gz
            mkdir -p /tmp/db2
            tar -xzf /tmp/db2.tar.gz -C /tmp/db2
            mkdir -p "$DEVSPACE_TOOLS/db2"
            mv /tmp/db2/clidriver "$DB2_HOME"
            rm -rf /tmp/db2 /tmp/db2.tar.gz
          fi

          "$JAVA_HOME/bin/keytool" -delete -noprompt \
            -alias company-ca \
            -keystore "$DEVSPACE_TOOLS/company-truststore.p12" \
            -storetype PKCS12 \
            -storepass changeit >/dev/null 2>&1 || true
          "$JAVA_HOME/bin/keytool" -importcert -noprompt \
            -alias company-ca \
            -file "$CORPORATE_CA" \
            -keystore "$DEVSPACE_TOOLS/company-truststore.p12" \
            -storetype PKCS12 \
            -storepass changeit >/dev/null

          cat > "$HOME/.m2/settings.xml" <<"EOF"
        <settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
                  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
          <mirrors>
            <mirror>
              <id>nexus</id>
              <url>https://nexus.example.com/repository/maven-public/</url>
              <mirrorOf>*</mirrorOf>
            </mirror>
          </mirrors>
          <servers>
            <server>
              <id>nexus</id>
              <username>${env.NEXUS_USERNAME}</username>
              <password>${env.NEXUS_PASSWORD}</password>
            </server>
          </servers>
        </settings>
        EOF

          cat > "$DB2DSDRIVER_CFG_PATH/db2dsdriver.cfg" <<EOF
        <?xml version="1.0" encoding="UTF-8" standalone="no" ?>
        <configuration>
          <dsncollection>
            <dsn alias="SALES_TLS" name="SALES" host="db2.internal.example.com" port="50001">
              <parameter name="Security" value="SSL" />
              <parameter name="SSLServerCertificate" value="$CORPORATE_CA" />
            </dsn>
          </dsncollection>
        </configuration>
        EOF

          cat > "$HOME/.local/bin/with-conjur" <<"EOF"
        #!/usr/bin/env sh
        set -eu
        token="$(curl --fail --silent --show-error --cacert "$CORPORATE_CA" \
          --request POST \
          --header "Authorization: Token token=\"$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\"" \
          --header "Accept-Encoding: base64" \
          "${CONJUR_AUTHN_URL}/${CONJUR_AUTHN_LOGIN}/authenticate")"
        secret() {
          curl --fail --silent --show-error --cacert "$CORPORATE_CA" \
            --header "Authorization: Token token=\"$token\"" \
            "${CONJUR_APPLIANCE_URL}/secrets/${CONJUR_ACCOUNT}/variable/$1"
        }
        export NEXUS_USERNAME="$(secret devspaces/nexus/username)"
        export NEXUS_PASSWORD="$(secret devspaces/nexus/password)"
        export DB2_USERNAME="$(secret devspaces/db2/username)"
        export DB2_PASSWORD="$(secret devspaces/db2/password)"
        unset token
        exec "$@"
        EOF
          chmod 0700 "$HOME/.local/bin/with-conjur"

          uv --version
          java -version
          mvn --version
          db2cli validate -help >/dev/null
        '
      group:
        kind: build
        isDefault: true
  - id: python-sync
    exec:
      component: rhel10
      workingDir: /projects/<repository-directory>
      commandLine: with-conjur uv sync --locked
      group:
        kind: build
  - id: maven-verify
    exec:
      component: rhel10
      workingDir: /projects/<repository-directory>
      commandLine: with-conjur mvn --batch-mode verify
      group:
        kind: build
  - id: db2-validate
    exec:
      component: rhel10
      workingDir: /projects/<repository-directory>
      commandLine: with-conjur db2cli validate -dsn SALES_TLS -connect -user "$DB2_USERNAME" -passwd "$DB2_PASSWORD"
events:
  postStart:
    - bootstrap-tools
```

The tool archive URLs, version numbers, Db2 hostname, Db2 database name, Conjur paths, and `CONJUR_AUTHN_LOGIN` are configuration values. Change them before creating the workspace. Pin artifact versions and have Nexus validate checksums before making them available to developers.

The `install` command used by the uv bootstrap must be present in the stock image. If it is absent, have the platform team use a stock RHEL 10 variant that includes coreutils; do not work around that by granting the workspace extra security context permissions.

## Configure Conjur and the workspace service account

Use the Conjur Kubernetes authenticator. Do not put a personal Conjur API key, a long-lived `CONJUR_AUTHN_TOKEN`, or Db2 credentials in the devfile or a Kubernetes Secret.

The OpenShift and CyberArk administrators must:

1. Create a dedicated service account, for example `devspace-db2`, in the workspace namespace.
2. Bind that service account to workspace pods with the DevWorkspace operator's approved namespace or controller policy.
3. Configure the Conjur Kubernetes authenticator with the OpenShift API endpoint and service-account issuer.
4. Create a Conjur host identity matching `CONJUR_AUTHN_LOGIN` and permit that identity to authenticate through the authenticator.
5. Permit that host identity to read only `devspaces/nexus/username`, `devspaces/nexus/password`, `devspaces/db2/username`, and `devspaces/db2/password`.

Example service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: devspace-db2
  namespace: <developer-namespace>
```

The `with-conjur` wrapper created by the devfile reads the mounted service-account JWT, receives a Conjur token, reads the four approved variables, and executes exactly one child command with the values in its environment. This limits credentials to the lifetime of `uv`, Maven, or Db2 command being run. It does not write passwords to the Git checkout, `settings.xml`, shell profile, or terminal history.

Confirm the exact Conjur authenticate URL, login identity, variable IDs, and API path format with the CyberArk team. Those paths differ with Conjur account and policy hierarchy.

## Configure Nexus behavior

The devfile uses Nexus in three places:

- Bootstrap archives are downloaded from a Nexus raw repository with `curl --cacert`.
- `UV_INDEX_URL` makes the internal Python simple index the default uv index.
- The generated Maven settings mirror every Maven repository request through `maven-public`.

Add `uv.toml` to the application repository so the lock file also records the intended default index:

```toml
[[index]]
name = "nexus"
url = "https://nexus.example.com/repository/pypi-internal/simple"
default = true
```

Commit `uv.lock` after dependency review. Run `with-conjur uv sync --locked` rather than `uv sync` for reproducible dependency resolution. Maven's `mirrorOf` value of `*` prevents Maven Central and any repository declared by a dependency from bypassing Nexus.

## Configure Db2

The `postStart` command generates a user-owned `db2dsdriver.cfg` with:

- `Security=SSL` to require TLS;
- `SSLServerCertificate` pointing to the repository CA bundle; and
- a `SALES_TLS` data source alias.

Update the alias, host, port, and database name in the devfile. The IBM driver version must be tested against the target Db2 server and installed from a Nexus repository that the organization is licensed to use.

The container environment provides the required Db2 values:

```sh
DB2_HOME=/tmp/devspace-home/.local/share/devspace-tools/db2/clidriver
DB2DSDRIVER_CFG_PATH=/tmp/devspace-home/.config/db2
LD_LIBRARY_PATH=/tmp/devspace-home/.local/share/devspace-tools/db2/clidriver/lib
```

Use the Dev Spaces command **db2-validate** or run the equivalent command from the terminal:

```sh
with-conjur db2cli validate -dsn SALES_TLS -connect \
  -user "$DB2_USERNAME" -passwd "$DB2_PASSWORD"
```

`DB2_USERNAME` and `DB2_PASSWORD` exist only in the subprocess started by `with-conjur`; they will not be available in the parent terminal. This is intentional.

For Java applications, retrieve the same values at application startup and use a JDBC URL such as:

```properties
db.url=jdbc:db2://db2.internal.example.com:50001/SALES:sslConnection=true;
```

Use the Db2 JDBC driver from the internal Maven group, not a local driver JAR copied into source control.

## Start the Dev Spaces workspace

1. Commit `devfile.yaml`, `certificates/company-ca.pem`, `uv.toml`, and the application source to the Git repository.
2. Open the OpenShift Dev Spaces dashboard and sign in with the organization identity provider.
3. Select **Create Workspace**, choose **Git Repository**, then enter the repository URL and the branch containing `devfile.yaml`.
4. Confirm the stock RHEL 10 image and start the workspace.
5. Wait for `bootstrap-tools` to complete, then open a terminal and validate the installed tools and TLS configuration:

```sh
id
cat /etc/redhat-release
uv --version
uv run python --version
java -version
mvn --version
db2cli validate -help
curl --fail https://nexus.example.com/service/rest/v1/status
curl --fail https://conjur.example.com/health
```

6. Run the Dev Spaces commands **python-sync**, **maven-verify**, and **db2-validate**. Each command obtains the Conjur values for its own process. The bootstrap installs Python 3.12 through the Nexus Python mirror; applications should use `uv run` or `uv python` to select that managed interpreter.

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Bootstrap stops before downloading tools | Stock RHEL image lacks `curl`, `tar`, `gzip`, or `install` | Use a stock RHEL 10 developer-capable image; do not add privileged SCC |
| TLS validation fails | CA file is missing, incomplete, or points at the wrong project path | Verify `CORPORATE_CA` and `openssl s_client` against the endpoint |
| `uv` contacts PyPI | Index URL or `uv.toml` points elsewhere | Check `UV_INDEX_URL` and run `uv sync --locked --verbose` |
| Maven contacts Central | Generated `settings.xml` was not created or uses a wrong mirror URL | Inspect `$HOME/.m2/settings.xml` and run `mvn --debug verify` through `with-conjur` |
| Conjur authentication is denied | Service account, host ID, authenticator, or Conjur policy differs from the configured values | Compare the mounted JWT identity with `CONJUR_AUTHN_LOGIN` and CyberArk policy |
| Db2 timeout or `SQL30081N` | Missing egress/DNS/firewall path or wrong endpoint | Check DNS, NetworkPolicy, Db2 firewall rules, host, and port |
| Db2 TLS error | The driver cannot validate the Db2 server certificate | Verify `Security=SSL`, `SSLServerCertificate`, and the CA chain |

## Security checklist

- Keep the workspace under the `restricted` SCC; do not grant privileged or `anyuid` access to install tools.
- Pin, checksum, scan, and retain the tool archives served by Nexus.
- Rotate Nexus and Db2 credentials in Conjur according to organization policy.
- Keep all credentials out of `devfile.yaml`, Git history, Maven settings, lock files, and persistent terminal profiles.
- Require TLS for Nexus, Conjur, and Db2. Do not add insecure trust bypasses.
- Restrict namespace egress to the approved Nexus, Conjur, and Db2 destinations.