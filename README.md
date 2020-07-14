## sdc-k8s-deployment-with-custom-config

This project provides examples and guidance for deploying custom configurations of [StreamSets Data Collector](https://streamsets.com/products/dataops-platform/data-collector) (SDC) on Kubernetes using [Control Hub](https://streamsets.com/products/dataops-platform/control-hub).  Each example provides an <code>sdc.yaml</code> that shows the relevant configuration.

Examples include:

* Example #1: Baked-in Configuration (and how to set Java Opts)
* Example #2: Ingress
* Example #3: Loading <code>sdc.properties</code> from a ConfigMap
* Example #4: Loading static and dynamic <code>sdc.properties</code> from separate ConfigMaps
* Example #5: Loading <code>credential-stores.properties</code> from a Secret
* Example #6: Loading resources a read-only Volume
* Example #7: Loading a keystore and a truststore from a single Secret



### Example 1: Baked-in Configuration
 
This approach packages a custom <code>sdc.properties</code> file within a custom SDC image at the time the image is built, similar to the example [here](https://github.com/onefoursix/control-agent-quickstart/tree/master/custom-datacollector-docker-image).  See the <code>Dockerfile</code> and <code>build.sh</code> artifacts in the [Example 1](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-1/sdc-docker-custom-config) directory.  

This approach is most suitable for [execution](https://streamsets.com/documentation/controlhub/latest/help/controlhub/UserGuide/DataCollectors/DataCollectors_title.html) SDCs whose config does not need to be dynamically set.  The <code>sdc.properties</code> file "baked in" to the custom SDC image may include custom settings for properties like <code>production.maxBatchSize</code> and email configuration if these properties are consistent across deployments. 

Set <code>http.realm.file.permission.check=false</code> in your <code>sdc.properties</code> file to avoid permission issues.

Java options, like heap size, can be set at deployment time by setting the <code>SDC_JAVA_OPTS</code> environment variable in the deployment manifest like this:

    env:
    - name: SDC_JAVA_OPTS
      value: "-Xmx4g -Xms4g"
      
See Example 1's [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-1/sdc.yaml) for the full deployment manifest.
 
### Example 2: Baked-in Configuration plus Ingress

Assuming an Ingress Controller is deployed, one can extend Example 1 (reusing the same custom SDC image with its baked-in <code>sdc.properties</code>) by adding a Service and Ingress to the SDC deployment manifest along with these two additional environment variables added to the deployment:
  
    - name: SDC_CONF_SDC_BASE_HTTP_URL
      value: https://<your ingress host>[:<your ingress port>]
    - name: SDC_CONF_HTTP_ENABLE_FORWARDED_REQUESTS
      value: true

One could bake these properties into a custom SDC image, as in Example 1, but that would limit the image's usefulness; typically, an SDC Base URL is set at deployment time.

See Example 2's [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-2/sdc.yaml) for the full deployment manifest.

#### A note about Environment variables
Environment variables with the prefix <code>"SDC_CONF_"</code>, like <code>SDC_CONF_SDC_BASE_HTTP_URL</code>, allow one to dynamically set properties in the deployed SDC's <code>sdc.properties</code> file. These environment variables are mapped to SDC properties by trimming the <code>SDC_CONF_</code> prefix, lowercasing the name and replacing <code>"_"</code> with <code>"."</code>.  So, for example, the value set in the environment variable <code>SDC_CONF_SDC_BASE_HTTP_URL</code> will be set in <code>sdc.properties</code> as the property <code>sdc.base.http.url</code>. 

However, this mechanism is not able to set mixed-case properties in <code>sdc.properties</code> like <code>production.maxBatchSize</code>.  If you want to set mixed-case properties then you either have to bake-in the settings in the <code>sdc.properties</code> file packaged in a custom SDC image (as in Example 1 above), or set the properties in a Volume mounted over the image's <code>sdc.properties</code> file as shown in Examples 3 and 4 below.

It's also worth noting that values for environment variables with the prefix <code>"SDC_CONF_"</code> are written to the <code>sdc.properties</code> file by the SDC container's <code>docker-entrypoint.sh</code> script which forces the SDC container to have read/write access to the <code>sdc.properties</code> file, which may not be the case if <code>sdc.properties</code> is mounted from a Volume.  

*Best practice is to load <code>sdc.properties</code> from a Volume and to avoid using <code>SDC_CONF_</code> environment variables.*

### Example 3: Loading <code>sdc.properties</code> from a ConfigMap

An approach that offers greater flexibility than the examples above is to dynamically mount an <code>sdc.properties</code> file at deployment time. One way to do that is to store an <code>sdc.properties</code> file in a configMap and to Volume Mount the configMap into the SDC container, overwriting the default <code>sdc.properties</code> file included with the image.

As mentioned above, when using this technique, the configMap's representation of <code>sdc.properties</code> will be read-only, so one can't use any <code>SDC_CONF_</code> prefixed environment variables in the SDC deployment; all custom property values for properties defined in <code>sdc.properties</code> need to be set in the  configMap (though one can still set <code>SDC_JAVA_OPTS</code> in the environment as that is a "pure" environment variable used by SDC).  

This example uses one monolithic <code>sdc.properties</code> file stored in a single configMap.  Start by copying a clean <code>sdc.properties</code> file to a local working directory. Set all property values you want for a given deployment.  For this example I will set custom values for these properties within the file (alongside all the other properties already in the file):

    sdc.base.http.url=https://sequoia.onefoursix.com
    http.enable.forwarded.requests=true
    http.realm.file.permission.check=false  # set this to avoid permission issues
    production.maxBatchSize=20000 
    
Save the edited <code>sdc.properties</code> file in a configMap named <code>sdc-properties</code> by executing a command like this:

    $ kubectl create configmap sdc-properties --from-file=sdc.properties

This configMap needs to be created in advance, outside of Control Hub, prior to starting the SDC deployment.

Add the configMap as a Volume in your SDC deployment manifest like this:

    volumes:
    - name: sdc-properties
      configMap:
        name: sdc-properties
        
Add a Volume Mount to the SDC container, to overwrite the <code>sdc.properties</code> file:

    volumeMounts:
    - name: sdc-properties
      mountPath: /etc/sdc/sdc.properties
      subPath: sdc.properties

See Example 3's [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-3/sdc.yaml) for the full deployment manifest.


### Example 4: Loading static and dynamic <code>sdc.properties</code> from separate ConfigMaps

This example splits the monolithic <code>sdc.properties</code> file used in Example 3 into two configMaps: one for properties loaded from a file that rarely if ever change (and that can be reused across multiple deployments), and one for dynamic properties targeted for a specific deployment that can be edited inline within a manifest.

Similar to Example 3, start by copying a clean <code>sdc.properties</code> file to a local working directory.

Comment out or delete the small number of properties that need to be set specifically for a deployment, leaving in place the majority of properties to be reused across deployments.  For example, I'll set and include these two properties in the file:

    http.realm.file.permission.check=false
    http.enable.forwarded.requests=true
    
But I will comment out these two properties which I want to set specifically for a given deployment:

    # sdc.base.http.url=http://<hostname>:<port>
    # production.maxBatchSize=1000
    
One final setting:  append the filename <code>sdc-dynamic.properties</code> to the <code>config.includes</code> property in the <code>sdc.properties</code> file like this:

    config.includes=dpm.properties,vault.properties,credential-stores.properties,sdc-dynamic.properties

That setting will load the dynamic properties described below.

Save the <code>sdc.properties</code> file in a configMap named <code>sdc-static-properties</code> by executing a command like this:

<code>$ kubectl create configmap sdc-static-properties --from-file=sdc.properties</code>

Once again, the configMap <code>sdc-static-properties</code> can be reused across multiple deployments.

Next, create a manifest named <code>sdc-dynamic-properties.yaml</code> that will contain only properties specific to a given deployment,  For example, my <code>sdc-dynamic-properties.yaml</code> contains  these two properties:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: sdc-dynamic-properties
    data:
      sdc-dynamic.properties: |
        sdc.base.http.url=https://portland.onefoursix.com
        production.maxBatchSize=20000
    
Create the configMap by executing a command like this:

<code>$ kubectl apply -f sdc-dynamic-properties.yaml</code>

Add two Volumes to your SDC deployment manifest like this:

    volumes:
    - name: sdc-static-properties
      configMap:
        name: sdc-static-properties
        items:
        - key: sdc.properties
          path: sdc.properties
    - name: sdc-dynamic-properties
      configMap:
        name: sdc-dynamic-properties
        items:
        - key: sdc-dynamic.properties
          path: sdc-dynamic.properties
        
And add two Volume Mounts to the SDC container, the first to overwrite the <code>sdc.properties</code> file and the second to add the referenced <code>sdc-dynamic.properties</code> file

    volumeMounts:
    - name: sdc-static-properties
      mountPath: /etc/sdc/sdc.properties
      subPath: sdc.properties
    - name: sdc-dynamic-properties
      mountPath: /etc/sdc/sdc-dynamic.properties
      subPath: sdc-dynamic.properties

See Example 4's [sdc-dynamic-properties.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-4/sdc-dynamic-properties.yaml) as well as [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-4/sdc.yaml) for the full deployment manifest 

### Example 5: Loading <code>credential-stores.properties</code> from a Secret

This example extends Example 4 by showing how to load a <code>credential-stores.properties</code> file from a Secret.  This technique is useful if you have different credential stores in different environments (for example, Dev, QA, Prod) and want each environment's SDCs to automatically load the appropriate settings.  

Start by creating a <code>credential-stores.properties</code> file.  For example, a <code>credential-stores.properties</code> file used for Azure Key Vault might look like this:

    credentialStores=azure
    credentialStore.azure.def=streamsets-datacollector-azure-keyvault-credentialstore-lib::com_streamsets_datacollector_credential_azure_keyvault_AzureKeyVaultCredentialStore
    credentialStore.azure.config.credential.refresh.millis=30000
    credentialStore.azure.config.credential.retry.millis=15000
    credentialStore.azure.config.vault.url=https://mykeyvault.vault.azure.net/
    credentialStore.azure.config.client.id=[redacted]
    credentialStore.azure.config.client.key=[redacted]
    
Store the <code>credential-stores.properties</code> file in a Secret; I'll name my secret <code>azure-key-vault-credential-store</code>:

    $ kubectl create secret generic azure-key-vault-credential-store --from-file=credential-stores.properties 

In your SDC deployment manifest, create a Volume for the Secret:

    volumes:
    - name: azure-key-vault-credential-store
      secret:
        secretName: azure-key-vault-credential-store
 
And then create a Volume Mount that overwrites the default <code>credential-stores.properties</code> file:

    volumeMounts:
    - name: azure-key-vault-credential-store
      mountPath: /etc/sdc/credential-stores.properties
      subPath: credential-stores.properties
         
See Example 5's [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-5/sdc.yaml) for the full deployment manifest.

### Example 6: Loading resources from a read-only Volume

This example showing how to load shared files and other resources from a read-only Volume.  This technique can be used to load any resources needed by SDC at deployment time that are not baked into the SDC image, including, for example, hadoop config files, lookup files, JDBC drivers, etc... 

See Example 6's [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-6/sdc.yaml) for an example deployment manifest.


### Example 7: Loading a keystore and a truststore from a single Secret

This example showing how to load a keystore and a truststore from a single Secret.

Start by creating a Secret that contains two files loaded from disk:

    $ kubectl create secret generic my-secrets \
       --from-file=my-truststore.jks \
       --from-file=my-keystore.jks 
    
In your SDC deployment manifest, create a Volume for the Secret with keys for both files:

    volumes:
    - name: my-secrets
      secret:
        secretName: my-secrets
        items:
        - key: my-keystore.jks
          path: my-keystore.jks
        - key: my-truststore.jks
          path: my-truststore.jks


In your SDC container, add two VolumeMounts that load both files from the Secret:

    volumeMounts:
    - name: my-secrets
      mountPath: /etc/sdc/my-keystore.jks
      subPath: my-keystore.jks
    - name: my-secrets
      mountPath: /etc/sdc/my-truststore.jks
      subPath: my-truststore.jks

Once the SDC container starts, <code>exec</code> into the container to see the two files loaded from the Secret:

    $ kubectl exec -it auth-sdc-84bdc7d58d-w8gt6 -- sh
    / $ ls -l /etc/sdc | grep "my-"
    -rw-r--r--    1 root     root            19 Jul 13 23:41 my-keystore.jks
    -rw-r--r--    1 root     root            21 Jul 13 23:41 my-truststore.jks

See Example 7's [sdc.yaml](https://github.com/onefoursix/sdc-k8s-deployment-with-custom-config/tree/master/Example-7/sdc.yaml) for an example deployment manifest.
