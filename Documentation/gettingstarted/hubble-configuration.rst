.. only:: not (epub or latex or html)

    WARNING: You are looking at unreleased Cilium documentation.
    Please use the official rendered version released here:
    https://docs.cilium.io

.. _hubble_configure:

********************
Hubble Configuration
********************

This page provides guidance to configure Hubble in a way that suits your
environment. Instructions to enable Hubble are provided as part of each
Cilium :ref:`getting_started` guide.

.. _hubble_configure_tls_certs:

TLS certificates
================

When Hubble Relay is deployed, Hubble listens on a TCP port on the host network.
This allows Hubble Relay to communicate with all Hubble instances in the
cluster. Connections between Hubble instances and Hubble Relay are secured using
mutual TLS (mTLS) by default.

TLS certificates can be provided by manually on the Helm install command (user
provided) or generate automatically via either:

* `Helm <https://helm.sh/docs/chart_template_guide/function_list/#gensignedcert>`__
* cilium's `certgen <https://github.com/cilium/certgen>`__ (using a Kubernetes ``CronJob``)
* `cert-manager <https://cert-manager.io/>`__

Auto generated certificates via Helm
------------------------------------

When using Helm, TLS certificates are (re-)generated every time Helm is used
for install or upgrade. As Hubble server and Hubble Relay support TLS
certificates hot reloading, including CA certificates, this does not disrupt
any existing connection. New connections are automatically established using
the new certificates without having to restart Hubble server or Hubble
Relay.

::

    --set hubble.tls.auto.enabled=true               # enable automatic TLS certificate generation
    --set hubble.tls.auto.method=helm                # auto generate certificates using helm method
    --set hubble.tls.auto.certValidityDuration=1095  # certificates validity duration in days (default 3 years)

The downside of the Helm method is that while certificates are automatically
generated, they are not automatically renewed.  Consequently, running
``helm upgrade`` is required when certificates are about to expire (i.e. before
the configured ``hubble.tls.auto.certValidityDuration``).

Auto generated certificates via certgen
---------------------------------------

Like the Helm method, certgen generates the TLS certificates at installation
time and a Kubernetes ``CronJob`` is scheduled to renew them (regardless of
their expiration date).

::

    --set hubble.tls.auto.enabled=true               # enable automatic TLS certificate generation
    --set hubble.tls.auto.method=cronJob             # auto generate certificates using cronJob method
    --set hubble.tls.auto.certValidityDuration=1095  # certificates validity duration in days (default 3 years)
    --set hubble.tls.auto.schedule="0 0 1 */4 *"     # schedule for certificates re-generation (crontab syntax)

Auto generated certificates via cert-manager
--------------------------------------------

This method relies on `cert-manager <https://cert-manager.io/>`__ to generate
the TLS certificates. cert-manager has becomes the de facto way to manage TLS on
Kubernetes, and it has the following advantages compared to the previously
documented methods:

* Support multiple issuers (e.g. a custom CA,
  `Vault <https://www.vaultproject.io/>`__,
  `Let's Encrypt <https://letsencrypt.org/>`__,
  `Google's Certificate Authority Service <https://cloud.google.com/certificate-authority-service>`__,
  and more) allowing to choose the issuer fitting your organization's
  requirements.
* Manages certificates via a
  `CRD <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/>`__
  which is easier to inspect with Kubernetes tools than PEM file.

**Installation steps**:

#. First, install `cert-manager <https://cert-manager.io/docs/installation/>`__
   and setup an `issuer <https://cert-manager.io/docs/configuration/>`_.
   Please make sure that your issuer is able to create certificates under the
   ``cilium.io`` domain name.
#. Install/upgrade Cilium including the following Helm flags:

::

    --set hubble.tls.auto.enabled=true               # enable automatic TLS certificate generation
    --set hubble.tls.auto.method=certmanager         # auto generate certificates using cert-manager
    --set hubble.tls.auto.certValidityDuration=1095  # certificates validity duration in days (default 3 years)
    --set hubble.tls.auto.certManagerIssuerRef.group="cert-manager.io" # Reference to cert-manager's issuer
    --set hubble.tls.auto.certManagerIssuerRef.kind="ClusterIssuer"
    --set hubble.tls.auto.certManagerIssuerRef.name="ca-issuer"

**Troubleshooting**:

While installing Cilium or cert-manager you may get the following error:

::

    Error: Internal error occurred: failed calling webhook "webhook.cert-manager.io": Post "https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=10s": dial tcp x.x.x.x:443: connect: connection refused

This happens when cert-manager's webhook (which is used to verify the
``Certificate``'s CRD resources) is not available. There are several ways to
resolve this issue. Pick one of the options below:

.. tabs::

    .. group-tab:: Install CRDs first

        Install cert-manager CRDs before Cilium and cert-manager (see `cert-manager's documentation about installing CRDs with kubectl <https://cert-manager.io/docs/installation/helm/#option-1-installing-crds-with-kubectl>`__):

        .. code-block:: shell-session

            $ kubectl create -f cert-manager.crds.yaml

        Then install cert-manager, configure an issuer, and install Cilium.

    .. group-tab:: Upgrade Cilium

        Upgrade Cilium from an installation with TLS disabled:

        .. code-block:: shell-session

            $ helm install cilium cilium/cilium \
                --set hubble.tls.enabled=false \
                ...

        Then install cert-manager, configure an issuer, and upgrade Cilium enabling TLS:

        .. code-block:: shell-session

            $ helm install cilium cilium/cilium --set hubble.tls.enabled=true

    .. group-tab:: Disable webhook

        Disable cert-manager validation (assuming Cilium is installed in the ``kube-system`` namespace):

        .. code-block:: shell-session

            $ kubectl label namespace kube-system cert-manager.io/disable-validation=true

        Then install Cilium, cert-manager, and configure an issuer.

    .. group-tab:: Host network webhook

        Configure cert-manager to expose its webhook within the host network namespace:

        .. code-block:: shell-session

            $ helm install cert-manager jetstack/cert-manager \
                    --set webhook.hostNetwork=true \
                    --set webhook.tolerations='["operator": "Exists"]'

        Then configure an issuer and install Cilium.

User provided certificates
--------------------------

In order to use custom TLS certificates, ``hubble.tls.auto.enabled`` must be
set to ``false``, secrets containing the certificates must be created in the
``kube-system`` namespace, and the secret names must be provided to Helm.

Provided files must be **base64 encoded** PEM certificates.

In addition, the **Common Name (CN)** and **Subject Alternative Name (SAN)**
of the certificate for Hubble server MUST be set to
``*.{cluster-name}.hubble-grpc.cilium.io`` where ``{cluster-name}`` is the
cluster name defined by ``cluster.name`` (defaults to ``default``).

Once the certificates have been issued, the secrets must be created in the ``kube-system`` namespace.

Each secret must contain the following keys:

- ``tls.crt``: The certificate file.
- ``tls.key``: The private key file.
- ``ca.crt``: The CA certificate file.

The following examples demonstrates how to create the secrets.

Create the hubble server certificate secret:

.. code-block:: shell-session

  $ kubectl -n kube-system create secret generic hubble-server-certs --from-file=hubble-server.crt --from-file=hubble-server.key --from-file=ca.crt

If hubble-relay is enabled, the following secrets must be created:

.. code-block:: shell-session

  $ kubectl -n kube-system create secret generic hubble-relay-server-certs --from-file=hubble-relay-server.crt --from-file=hubble-relay-server.key --from-file=ca.crt
  $ kubectl -n kube-system create secret generic hubble-relay-client-certs --from-file=hubble-relay-client.crt --from-file=hubble-relay-client.key --from-file=ca.crt

If hubble-ui is enabled, the following secret must be created:

.. code-block:: shell-session

  $ kubectl -n kube-system create secret generic hubble-ui-client-certs --from-file=hubble-ui-client.crt --from-file=hubble-ui-client.key --from-file=ca.crt

Lastly, if the Hubble metrics API is enabled, the following secret must be created:

.. code-block:: shell-session

  $ kubectl -n kube-system create secret generic hubble-metrics-certs --from-file=hubble-metrics.crt --from-file=hubble-metrics.key --from-file=ca.crt

After the secrets have been created, the secret names must be provided to Helm and automatic certificate generation must be disabled:

::

    --set hubble.tls.auto.enabled=false                                       # Disable automatic TLS certificate generation
    --set hubble.tls.server.existingSecret="hubble-server-certs"
    --set hubble.relay.tls.server.enabled=true                                # Enable TLS on Hubble Relay (optional)
    --set hubble.relay.tls.server.existingSecret="hubble-relay-server-certs"
    --set hubble.relay.tls.client.existingSecret="hubble-relay-client-certs"
    --set hubble.ui.tls.client.existingSecret="hubble-ui-client-certs"
    --set hubble.metrics.tls.enabled=true                                     # Enable TLS on the Hubble metrics API (optional)
    --set hubble.metrics.tls.server.existingSecret="hubble-metrics-certs"

- ``hubble.relay.tls.server.existingSecret`` and ``hubble.ui.tls.client.existingSecret``
  only need to be provided when ``hubble.relay.tls.server.enabled=true`` (default ``false``).
- ``hubble.ui.tls.client.existingSecret`` only needs to be provided when ``hubble.ui.enabled`` (default ``false``).
- ``hubble.metrics.tls.server.existingSecret`` only needs to be provided when ``hubble.metrics.tls.enabled`` (default ``false``).
  For more details on configuring the Hubble metrics API with TLS, see :ref:`hubble_configure_metrics_tls`.

.. _hubble_configure_metrics_tls:

Metrics TLS and Authentication
===============================

Starting with Cilium 1.16, Hubble supports configuring TLS on the Hubble
metrics API in addition to the Hubble observer API.

This can be done by specifying the following options to Helm at install or
upgrade time, along with the TLS configuration options described in the
previous section.

To enable TLS on the Hubble metrics API, add the following Helm flag to your
list of options:

::

    --set hubble.metrics.tls.enabled=true # Enable TLS on the Hubble metrics API

If you also want to enable authentication using mTLS on the Hubble metrics API,
first create a ConfigMap with a CA certificate to use for verifying client
certificates:

::

    kubectl -n kube-system create configmap hubble-metrics-ca --from-file=ca.crt

Then, add the following flags to your Helm command to enable mTLS:

::

    --set hubble.metrics.tls.enabled=true                       # Enable TLS on the Hubble metrics API
    --set hubble.metrics.tls.server.mtls.enabled=true           # Enable mTLS authentication on the Hubble metrics API
    --set hubble.metrics.tls.server.mtls.name=hubble-metrics-ca # Use the CA certificate from the ConfigMap

After the configuration is applied, clients will be required to authenticate
using a certificate signed by the configured CA certificate to access the
Hubble metrics API.

.. note::

  When using TLS with the Hubble metrics API you will need to update your
  Prometheus scrape configuration to use HTTPS by setting a ``tls_config`` and
  provide the path to the CA certificate. When using mTLS you will also need to
  provide a client certificate and key signed by the CA certificate for
  Prometheus to authenticate to the Hubble metrics API.
