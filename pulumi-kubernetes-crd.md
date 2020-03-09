# Manage a Kubernetes Custom Resource Definition (CRD) with Pulumi

Cloud providers of Kubernetes clusters frequently support Kubernetes Custom Resource Definitions (CRDs).

For example, Google Cloud lets you create managed SSL certificates by creating a ManagedCertificate,
or configure the Identity-Aware Proxy for a service via a BackendConfig.

Pulumi lets you "implement" type-safe CRDs by extending a CustomResource. Here's our BackendConfig:

```ts
// backend-config.ts

import * as kubernetes from '@pulumi/kubernetes';
import { CustomResourceOptions, Output } from '@pulumi/pulumi';

// Handwritten type
type BackendConfigArgs = {
  spec: {
    iap: {
      enabled: boolean;
      oauthclientCredentials: {
        secretName: Output<string>;
      };
    };
  };
  name: string;
  // Kubernetes namespace of the resource.
  namespace?: Output<string>;
};

// Extend kubernetes.apiextensions.CustomResource
export class BackendConfig extends kubernetes.apiextensions.CustomResource {
  // Match the constructor of any other resource
  constructor(
    resourceName: string,
    { name, spec, namespace }: BackendConfigArgs,
    opts: CustomResourceOptions,
  ) {
    // Merge whatever inputs (args) you take with constant values for the CRD
    const args: kubernetes.apiextensions.CustomResourceArgs = {
      apiVersion: 'cloud.google.com/v1beta1',
      kind: 'BackendConfig',
      metadata: { annotations: {}, name, namespace },
      spec,
    };

    // Finally, create the resource
    super(resourceName, args, opts);
  }
}
```

You can use this as you would any other Pulumi resource:

```ts
import { BackendConfig } from './backend-config';

// Assuming these resources are already created
const monitoringNamespace = …;
const polarisOauthclientCredentials = …;

const polarisBackendConfig = new BackendConfig(
  // Resource name, as usual
  'polaris_backend_config',

  // The args specified in the custom type
  {
    name: 'polaris-backend-config',
    namespace: monitoringNamespace.metadata.name,
    spec: {
      iap: {
        enabled: true,
        oauthclientCredentials: {
          secretName: polarisOauthclientCredentials.metadata.name,
        },
      },
    },
  },

  // Any options, such as `provider` or `protect`
  { protect: true }
);
```
