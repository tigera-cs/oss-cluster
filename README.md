# OSS Example Policy Design

## Label design

Even when they are useful to capture some needed metadata within the pods, it is advisable not to assign too many labels to pods in advance, and add them later only if needed. This is  because labels must have a specific meaning, and removing labels can have an unintended negative effect if those are not thoroughly understood. 

The labels referred in this document take into account characteristics of the environment which can be taken into consideration within a network security policy context. Other labels used explicitly for scheduling, identifying critical workloads, or other purposes are not considered. There will be cases where a unique label fits several purposes, for example “kubernetes.io/os: linux” can be used to avoid running windows pods in Linux nodes, but can be referenced in host network policies within rules which address specific security vulnerabilities regarding Linux nodes. The same applies to labels identifying zones or regions because apart from their intuitive use on scheduling, the final goal could be restricting the traffic between different geographic locations.

## Label scope:
Before detailing the different label guidelines which could apply to the use case described in the previous section, as labels are a construct which provide additional metadata when associated to different objects within kubernetes, it is important to define the scopes within where the labels will be applied:

- **Namespace:** Labels applied to the namespaces will be enforced on every pod running in such namespaces. As only the platform team will be creating namespaces for the different applications, Role-based access control  can be leveraged to achieve the needed security compliance.
- **Pod:** Labels at the pod level will be used by the applications team to define specific micro-segmentation controls within the context of one or multiple namespaces.
- **NetworkSet:** A network set is a namespaced resource used for defining IP ranges external to the Kubernetes cluster subnets. They contain services which are accessed by the pods within a specific namespace, or define the source range of clients (or proxies for such clients) connecting to the pods in a specific namespace from outside.
- **GlobalNetworkSet:** Shares the same functionality of the NetworkSets described above, but are referenced by pods no matter on which namespace they are located, so they must be used for well-known sources or destinations to the whole Kubernetes cluster. In the rest of this document, unless explicitly indicated, we will be using the wording “Calico NetworkSets” when referring to the namespaced resource NetworkSet, or GlobalNetworkSet interchangeably.

## Label domains:

In the following paragraphs, we expose the different context or domains the labels will cover regarding the security requirements exposed in the previous section:

- Ingress and egress access (North-South perimeter communication, the - - Kubernetes environment)
- Multi-tenancy
- Application microsegmentation
- Host endpoints (k8s rules for the Kubernetes nodes themselves)
- Ingress and egress access:

Ingress/Egress labels can be used to identify hosts or “external endpoints” to the cluster (different from pods, or Kubernetes nodes) using Calico NetworkSets associated with the source or destination IPs of such external hosts. Those Calico NetworkSets can then be selected as source or destination of the traffic within the network policies applied to the endpoints (pods or k8s nodes). 

Typical Calico NetworkSets used on such policies define the range of IPs used by the ingress controllers or load balancers (for the incoming traffic), or external services as APIs, shared resources, or repos. Examples of those labels will be:

```
eep.<organisation_domain>/ingress-lb: <tenant-id-load-balancer>
eep.<organisation_domain>/egress-api: <tenant-id-api>
eep.<organisation_domain>/egress-db: <tenant-id-db>
```
> eep is used as prefix for “external endpoint”

The creation of NetworkSets (if the source/destinations they define are restricted to a namespace), or a GlobalNetworkSets (if they affect potentially all pods within the environment across several namespaces) can rely on RBAC controls too, so only platform team members can create namespaced or global Calico NetworkSets to define the allowed traffic patterns within the environment that eventually are defined as source or destination within the network policies applied to objects having the labels above.

## Multi-tenancy:

Multi-tenancy labels cover both use cases, specific tenants, and shared services. Those endpoints are then selected within network policies when defining the East-West or North-South controls for the tenants. The recommendation is to attach multi-tenancy labels to both, pods and namespaces. Examples are:

```
sec.<organisation_domain>/tenant-id: <tenant-uid>
sec.<organisation_domain>/environment: <dev|test|prod>
sec.<organisation_domain>/compliance: <compliance-requirement-uid>
```
> sec is used to identify a secured environment, and used as prefix for labels within a tenant context

## Application Micro Segmentation:

Labels within an application micro-segmentation context can be used to identify microservices within a given application. Thus, endpoints are selected within network policies when defining East-West and North-South controls for microservice communication. 

In most of the cases, one application per namespace usually meets the security requirements, with perhaps the exception to shared services, which are deployed in their own namespaces in order to be accessed by several tenants. The recommendation is attaching application micro-segmentation labels to pods. 

These labels typically group pods running different microservices (i.e. frontend, db, etc), so there will be a label that groups them within the application (i.e. “name: app1”), and their specific function (i.e. “type: database”). Examples of micro segmentation labels are:

```
app.<organisation_domain>/name: <app-uid>
app.<organisation_domain>/type: <app-type-uid>
app.<organisation_domain>/version: <app-version>
app.<organisation_domain>/service: <svc-uid>
app.<organisation_domain>/tier: <app-tier> 
```
> app is used as prefix for labels within a microsegmentation context

## Host Endpoints:

Host endpoint labels can be used for Kubernetes nodes, or bare-metal hosts in case they must be integrated too. Once Host endpoint protection is enabled, such endpoints can then be selected in network policies the same way as pods, and the following controls can be implemented:

- **Cluster hosts:** Controls from-and-to hosts, or pods which connect to the k8s host network namespace (as calico system pods). This is a requirement to secure the cluster, and implement control-plane, and management-plane security controls.

- **External hosts:** Security controls for legacy applications hosted on bare-metal or virtual machines which typically will be used during integration, or migration scenarios.

Common examples of host endpoint labels are:
```
kubernetes.io/arch: amd64
kubernetes.io/os: linux
node-role.kubernetes.io/worker: true
node-role.kubernetes.io/egress-gateway: true
topology.kubernetes.io/region: ca-central-1
topology.kubernetes.io/zone: ca-central-1a
```

## Tenant & Namespace Design
Namespaces represent the application trust boundaries. We would recommend a Namespace per application. It is crucial to have a name standard for Namespaces that would reflect relevant information. The recommended Namespace standard includes information about tenant-id, tenant-team, app-id, app-env: 

```
AAAA-BBBB-CCCC-DDDDD-EEE 
```

Where: 
- **AAAA:** tenant-id acronym 
- **BBBB:** tenant-team acronym 
- **CCCC:** dev/stg/prod (a 3-digit acronym of app environment) 
- **DDDDD:** application name 
- **EEE:** application environment (dev, stg, prd) 

You can choose a different length based on your preference, as long as the acronyms are simple and relevant. 

## Example Namespaces

The following table is an example of how to map out namespaces based on tenant, team, IP Pool, environment, application, and application environment. For tenant team, environment, and application name, there are shorthand columns for the abbreviations as shown above that are used in the namespace name.

| Tenant-ID | Tenant Team    | Team ID | IP Pool                             | Environment     | Environment ID | Application Name | Application ID | Application Env | Namespace Name           |
|-----------|----------------|---------|-------------------------------------|-----------------|----------------|------------------|----------------|-----------------|--------------------------|
| T001      | Web Frontend   | webf    | int-dev-pool-10.21.0.0-18           | int-dev         | idev           | webstore         | webst          | dev             | t001-webf-idev-webst-dev |
| T002      | Authentication | auth    | int-dev-pool-10.21.0.0-19           | int-dev         | idev           | auth             | auth           | stg             | t002-auth-idev-auth-stg  |
| T003      | Payments       | paym    | conf-dev-pool-10.21.64.0-18         | conf-dev        | cdev           | payment          | paym           | prd             | t003-paym-cdev-paym-prd  |
| T900      | Monitoring     | mntr    | shared-services-pool-10.21.192.0-18 | shared-services | ssvc           | thanos           | thanos         | prd             | t900-mntr-ssvc-mntr-prd  |


## Example Labelling Standard
Here are some examples of labels that should be applied to both namespaces and applications.

> Note: For all examples going forward, <organisation_domain> in any example labels will be replaced with kubernetes.io.

**Namespace Labelling:**
```
sec.kubernetes.io/tenant-id: <tenant-uid>
sec.kubernetes.io/environment: <dev|test|prod>
sec.kubernetes.io/compliance: <compliance-requirement-uid>
```


**Application Labelling:**
```
app.kubernetes.io/name: <app-uid>
app.kubernetes.io/type: <app-type-uid>
app.kubernetes.io/version: <app-version>
app.kubernetes.io/service: <svc-uid>
app.kubernetes.io/tier: <app-tier> 
app.kubernetes.io/ingress-access: <True|False>
```

The above labels are used to identify the applications, their version, type, unique services within the application, the tier of the application and if ingress-access is needed for the application.
IP Pool Design

For the IP Pools, each pool will be designated to a functional area such as Internal Dev, Conf Dev or Shared Services. This will follow the naming convention outlined above in the Labelling Standards Section. These IP Pools will fall within the network that is assigned to the cluster Pod CIDR.

## Example IP Pool Configuration:

**Cluster Pod CIDR: 10.21.0.0/16**
```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: int-dev-pool-10.21.0.0-18
spec:
 cidr: 10.21.0.0/18
 blockSize: 26
 natOutgoing: true
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: conf-dev-pool-10.21.64.0-18
spec:
 cidr: 10.21.64.0/24
 blockSize: 26
 natOutgoing: true
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: shared-services-pool-10.21.192.0-18
spec:
 cidr: 10.21.192.0/24
 blockSize: 26
 natOutgoing: true
```

## Namespace Design

Namespaces will be created for each application in the cluster, these namespaces will follow the above naming conventions and include labels to assign them to the proper IP Pools. The namespaces will use common Kubernetes labels with the prefix sec.kubernetes.io to achieve the proper assignments.

Below is an example of a namespace along with the associated labels that resides in tenant 't001', in the 'int-dev' environment with no special compliance requirements. The name is from the above-mentioned formula, in this example the namespace has the following properties:


| **Namespace Property**      | Value        | Short ID |
|-----------------------------|--------------|----------|
| **Tenant-ID**               | Tenant 1     | t001     |
| **Tenant Team**             | Web Frontend | webf     |
| **Environment**             | Internal Dev | idev     |
| **Application Name**        | Web store    | webst    |
| **Application Environment** | Production   | prd      |


This results in a namespace name of **t001-webf-idev-webst-prd**.

**Namespace Example Manifest:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
 name: t001-webf-idev-webst-prd
 labels:
   sec.kubernetes.io/tenant-id: t001
   sec.kubernetes.io/environment: int-dev
   sec.kubernetes.io/compliance: none
 annotations:
   cni.projectcalico.org/ipv4pools: '["int-dev-pool-10.21.0.0-18"]'
```

Using the cni.projectcalico.org/ipv4pools annotation will ensure that the namespace only uses addresses from the desired pool.

## Global Labels
Along with the labels used for the namespaces, additional labels will need to be added to workloads to ensure that they segregated properly by the configured security policies.

**GlobalNetworkSet Labels**

Labels on the GlobalNetworkSets will be used in policies to identify trusted sources, the following is an example of a GlobalNetworkSet with a standard eep.kubernetes.io label applied (External End Point). The label value of ‘int-dev-t001-destinations’ indicated that this is a group of destinations that will be allowed from tenant 't001' in 'int-dev'. Again, this will use the standard Kubernetes prefix of eep.kubernetes.io.

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
 name: int-dev-t001-destinations
 labels:
   eep.kubernetes.io/trusted-destination: "int-dev-t001-destinations"
spec:
 nets:
   - 172.16.100.0/24
   - 192.168.1.0/24
   - 8.8.8.8/32
```

**GlobalNetworkSets to Include:**
The following GlobalNetworkSets should include the following addresses and their associated addresses.

| Label                                 | Description                                                                                                                               |
|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| eep.kubernetes.io/trusted-destination | Trusted external endpoint destinations for each namespace.                                                                                |
| eep.kubernetes.io/load-balancer       | Trusted load balancer for the tenant/namespace/application. This will depend on how the load balancers will be separated for the tenants. |
| eep.kubernetes.io/trusted-source      | Trusted sources for ingress rules to a namespace/tenant/host endpoint.                                                                    |


## Security Policy Design

Security Policies in the cluster will be created at different levels depending on the scope that they apply to. Global Security Policies have the largest scope, Tenant Level Policies are restricted to only the Tenant and the Application level policies will apply only at the application level.

All Security Policies will also be assigned an order which helps with the order in which they are processed, the follow is a rough guide of where each type of policy will fit into the order:

**Security Policy Order Guidelines**

Below is a table of guidelines to follow when creating the order of Security Policies. Policies will be evaluated in order as traffic is sent and received.

| Order Value | Type                | Function                                        | Namespace (if applicable) |
|-------------|---------------------|-------------------------------------------------|---------------------------|
| 1-100       | GlobalNetworkPolicy | Core Services                                   | n/a                       |
| 102-1000    | GlobalNetworkPolicy | Tenant Segregation                              | n/a                       |
| 1000-999998 | NetworkPolicy       | Namespace Controls & Optional Microsegmentation | Yes                       |
| 999999      | GlobalNetworkPolicy | Default Deny                                    | n/a                       |



### Global Security Policies

Global Security Policies are cluster level policies that allow or deny traffic globally in the cluster, they are designed for controlling traffic at the platform level. The most common core policy is to allow traffic to and from the cluster DNS service. Below is the example policy to allow this, this policy is also number one in the policy order:

**Ingress Security Policy**

Allow Ingress kube-dns Manifest:
```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: ingress-kubedns
spec:
  order: 1
  selector: k8s-app == "kube-dns"
  types:
    - Ingress
  ingress:
    - action: Allow
      destination:
        ports:
          - 53
      protocol: TCP
    - action: Allow
      destination:
        ports:
          - 53
      protocol: UDP
```

Any other core services should be allowed by similar policies with incrementing order values.

**Egress Security Policy**

Egress traffic will be handled at the Tenant and Application level, so no matching egress Global Security Policy will be created for the core services.

**Default Deny**

Finally, a global default-deny should be created. Note that this policy will deny traffic that hasn’t been explicitly allowed, and only should be applied after the rest of the policies have been created and applied. It is designed to come last in the GlobalNetworkPolicy order by giving it order number 999999. To avoid breaking vital communication in the cluster, the default deny is using a selector to exclude the 'kube-system', 'calico-system', and 'calico-apiserver' namespaces. Another failsafe that is built in is allowing traffic to the 'kube-dns' application.
Default Deny Manifest:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
 name: default-deny-policy
spec:
 order: 99999
 namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system", "calico-apiserver"}
 types:
   - Ingress
   - Egress
 egress:
   - action: Allow
     protocol: UDP
     destination:
       selector: k8s-app == "kube-dns"
       ports:
         - 53
```

### Tenant Security Policies

Tenant Level Security Policies will apply at the Global level, but by using label selection, will restrict access between Tenants and Development Environments. To accomplish this, each Namespace must be properly labelled with the Tenant and Environment, as these will be the key selectors in the policy.

These templates can be used for each Tenant and Environment combination and customized based on the ingress and egress requirements.

**Ingress Security Policy**

In the below ingress example, the Security Policy will select endpoints in the 't001' Tenant, in the 'int-dev' Environment with an application label of app.kubernetes.io/ingress-access == "true".  The 

This policy allows traffic from the GlobalNetworkSet with label eep.kubernetes.io/load-balancer == "t001-lb”. As previously stated, how the load balancers are segregated will determine if there are individual load balancers per tenant or if they are shared more broadly.

**Example Tenant Ingress Security Policy:**

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
 name: ingress-int-dev-t001
spec:
 order: 101
 namespaceSelector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev" && app.kubernetes.io/ingress-access == "true")
 selector: app.kubernetes.io/ingress-access == "true"
 types:
   - Ingress
 ingress:
   - action: Allow
     source:
       selector: eep.kubernetes.io/load-balancer == "t001-lb"
```


**Egress Security Policy**

In the below egress example, the Security Policy will select endpoints in the same Tenant and Environment as above, with the ingress-access label omitted.

This policy will allow egress traffic to the kube-dns service as well as to other endpoints within the 't001' Tenant, in the 'int-dev' Environment. It finally allows outbound traffic to the GlobalNetworkSet with label eep.kubernetes.io/trusted-destination == "int-dev-t001-destinations".

**Example Tenant Egress Security Policy:**

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: egress-int-dev-t001
spec:
  order: 102
  namespaceSelector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev")
  types:
    - Egress
  egress:
    - action: Allow
      destination:
        selector: k8s-app == "kube-dns"
    - action: Allow
      destination:
        namespaceSelector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev")
    - action: Allow
      destination:
        selector: eep.kubernetes.io/trusted-destination == "int-dev-t001-destinations"
```

### Application Security Policies

Application Security Policies will apply at the Namespace level, as each Namespace should represent a single application in the cluster. Again, by using label selection the application will be identified based on the Tenant, Environment, and Application name. Optionally, at this level, micro-segmentation can be implemented by including the Service.

The templates here can be customized per application at initial deployment time.

**Ingress Security Policy**

In the below ingress example, the Security Policy will apply only to the 't001-webf-idev-webst' namespace and select endpoints from 't001' in the 'int-dev' environment.

This Security Policy allows traffic from the same namespace, tenant, environment, and application to any of the matched endpoints.

Example Application Ingress Security Policy:
```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
 name: ingress-t001-webf-idev-webst-dev
 namespace: t001-webf-idev-webst-dev
spec:
 order: 1001
 types:
   - Ingress
 ingress:
   - action: Allow
     source:
       selector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev")
```

**Egress Security Policy**

In the below egress example, the Security Policy allows traffic from endpoints in the same namespace, tenant, and environment.

Example Application Egress Security Policy:
```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
 name: egress-t001-webf-idev-webst-dev
 namespace: t001-webf-idev-webst-dev
spec:
 order: 1002
 selector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev")
 types:
   - Egress
 egress:
   - action: Allow
     destination:
       selector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev")
```


### Application Micro Segmentation (Optional)

As mentioned above, at the namespace level, application micro segmentation can also be implemented. To accomplish this, each application service will need to be labelled appropriately, and then the following policies can be used instead of the above application labels.

The templates here can be customized per application service at initial deployment time.

**Ingress Security Policy**

In the below ingress example, the Security Policy will apply only to the 't001-webf-idev-webst' namespace and select endpoints from 't001' in the 'int-dev' environment with endpoints labelled with 'app1' and 'svc1'.

It allows traffic from the same namespace, tenant, environment, and application, but, with the service label of 'svc2' that is destined for 'svc1' on TCP port 10012. For these policies, the traffic will be selected based on source and destination selectors in the policy to achieve fine grain control.

For providing micro segmentation controls on the application, only ingress Security Policies will be used to restrict incoming traffic to each service.

Example Application Service Ingress Security Policy:

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
 name: ingress-t001-webf-idev-webst-dev-svc1
 namespace: t001-webf-idev-webst-dev
spec:
 order: 1001
 selector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev" && app.kubernetes.io/name == "app1" && app.kubernetes.io/svc == "svc2")
 types:
   - Ingress
 ingress:
   - action: Allow
     source:
       selector: (sec.kubernetes.io/tenant == "t001" && sec.kubernetes.io/env == "int-dev" && app.kubernetes.io/name == "app1" && app.kubernetes.io/svc == "svc1")
     destination:
       ports: 10012
     protocol: TCP
```

