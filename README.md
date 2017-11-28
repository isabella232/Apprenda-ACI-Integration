# ACI Networking Integration
![Automatic ACI Configuration](/images/automatic-aci-configuration.png)

This integration uses ACI to provide automatic application-level network isolation for Kubernetes workloads deployed with Apprenda.

### Supported Software Versions
|Software|Min Version|Max Version|
|-|-|-|
|Apprenda|7.0|8.0|
|Kubernetes|1.6.0|1.6.12|
|ACC|1.6.0|1.6.0|
|ACI|3.0|3.0|

## Prerequisites
* A functional Apprenda instance running on ESXi hosts
* An ACI environment configured for use with the ESXi hosts
* The ability to access the developer portal from all Windows nodes

### Kubernetes Installation with the ACI CNI
*We'll be using the [Kismatic Enterprise Toolkit (KET)](https://github.com/apprenda/kismatic) for installing Kubernetes*
1. Follow steps provided in the [guide](https://github.com/apprenda/kismatic/blob/master/docs/install.md) in order to generate your kismatic-cluster.yaml
2. To specify a custom CNI provider for KET, modify your kismatic-cluster.yaml and change the provider to "custom" as shown below:
      ```
      add_ons:
        cni:
          disable: false
          provider: custom
      ```
3. Install Kubernetes using the following command:
   ```
   kismatic install apply
   ```
4. You must install the ACI CNI manually by following the installation steps [here](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/kb/b_Kubernetes_Integration_with_ACI.html)

## Installation
1. Create a domain account for the integration
2. Run GenerateCertificate.ps1 to generate the credentials-encrypting certificate
3. Copy APICProxy.pfx (created by step 2) and ProvisionNode.ps1 to each Windows node
4. Run ProvisionNode.ps1 on each Windows node
    * This will import the APICProxy certificate, grant the domain account read access to the private key, and grant "Log on as service" to the domain account
5. Run CustomizeArchive.ps1 to embed the domain credentials in the archive
6. Copy ApicProxy.zip, ApicProxy.Bootstrapper.zip, and ConfigureApprenda.ps1 to the LM
7. Run ConfigureApprenda.ps1 on the LM (Apprenda SDK required)
    * This will create and promote the APIC Proxy application, create all the necessary Custom Properties, and create the APIC Proxy Bootstrapper (it will also set credentials if specified)
    
## Verification
1. Verifiy that credentials are set by launching the APIC Proxy UI
2. Create a new application using the provided connectivity-pod.yaml
3. Add "ping/unspecified/icmp" and "web/3000/tcp" to the "Application Ports" component custom property
4. Set the "Apprenda Network" component custom property to "Development Team"
5. Promote the application to sandbox
    * Use APIC to verify that an ANP, EPG, contract, and filter were created
    * Verify that the Connectivity Pod web UI can be accessed by launching the application from the developer portal
    * Use the Connectivity Pod web UI to verify connectivity with external resources
6. Demote the application
    * Use APIC to verify that the ANP, EPG, contract, and filter were removed

## Usage
![Component Custom Properties](/images/component-custom-properties.png)

The integration provides the following kubernetes-specific component custom properties to developers. These custom properties direct automatic ACI configuration on application promotion. ACI objects created by the integration will automatically be removed on application demotion or deletion. An application must be promoted for any changes to the custom properties to take effect.
1. **APIC Tenant** - Used for specifying the tenant to configure in APIC. This must coincide with the cluster that your application will be deployed to.
2. **Application Ports** - Used for specifying the set of ports and protocols over which your application will accept traffic. Each entry must be supplied in the format: name/port/protocol. "unspecified" may be used as a wildcard the port and/or protocol.
3. **Apprenda Network** - Used for specifying the group of applications that will be allowed access to your application. Selecting "None" will result in the application being added to the "kube-default" EPG.
4. **Consumed Contracts** - Used for specifying existing contracts (services) that your application should consume. You may specify any contract residing in the specified APIC tenant or in the common tenant.
5. **Provided Contracts** - Used for specifying existing contracts (services) that your application should provide. You may specify any contract residing in the specified APIC tenant or in the common tenant.

## Troubleshooting
The log can be accessed by launching the application and navigating to /log.
* Failure to save credentials
  1. Check that all Windows nodes have the "APICProxy" certificate installed under LocalMachine/Personal/Certificates
* Failure to promote
  1. Check that all Windows nodes can access the developer portal
  2. Check that correct service URLs and credentials are configured
  3. Check that all Windows nodes have the APICProxy certificate installed under LocalMachine/Personal/Certificates and that the domain account has been granted read access to the private key
  4. Check that the ACI CNI has been correctly configured for the ACI environment
* Failure to remove ACI resources on demote
  1. Check that the workload had been removed from Kubernetes
  2. Check that all Windows nodes have the APICProxy certificate installed under LocalMachine/Personal/Certificates and that the domain account has been granted read access to the private key
  3. Check that the domain account has been granted "Log on as service" in Local Group Policy on all Windows nodes