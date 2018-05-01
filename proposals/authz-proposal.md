# Kubeflow User Authorization

**Lifecycle:** **draft** | approved | implemented | verified | obsolete | rejected

**Authors:**



## Motivation
Allow users and RBAC rules for a kubeflow deployment to be defined by a CRD. This CRD can generated manually or by a provider.

### Goals

- Define templates for cluster level roles for Admin, Write and Read. Create these roles on the cluster as part of boot-strapping the cluster.
 - Admin - has full access to the namespace. Note this is namespace admin and not cluster admin.
 - Write - can make changes to the k8s resources in the specified namespace. This includes reading and listing the              resources.
 - Read- can read the resources in the specified namespace
 - None - have no access in the namespace. This is the default role.

- Configure rolebindings for the namespace by using GitHub team names and assigning specific roles to each team. There are    four roles in a namespace.

 - Configure rolebindings for the namespace based on a repo in GitHub for the org. Teams and their permissions are automatically obtained from GitHub repo.

 - Configure rolebindings for the namespace based on YAML directly provided by the cluster operator.

 - List all the teams and their permissions in the GitHub org

 - List all the repos in the GitHub org

 - List all the teams and their permissions in the repo

 - Synchronize the permissions on the namespace with the permissions on the GitHub repo or using the provided YAML file for the org


### Non-goals
- User authentication using GitHub. The proposal focuses on user authorization only.
- RBAC configuration of service accounts. The proposal focuses on end user RBAC configuration only.
- PV and PVC bindings. PV are cluster level resources. PV to PVC bindings is done in a separate proposal.

### User stories
- I am a data scientist and part of the GitHub org which represents my organization. I would like to create a namespace for my ML project and inherit users from GitHub teams and their associated privileges. I should be granted admin privileges for this new namespace if it is successfully created.
| `kubectl create managedNamespace source=GitHub, org=AIPG, Team=Algo:Role=Write, Team=BD:Role=read `|

  - I should receive a error message if namespace already exists, team does not exist in GitHub org or I specify an unsupported role (Admin, Write and Read) for the team.

- I am a data scientist and part of the GitHub org which represents my organization. I would like to create a namespace and inherit users and their roles from a repo in my GitHub org.
  - I should receive a error message if namespace already exists on the cluster or the repo does not exist in my GitHub org.

- I am a data scientist and part of the GitHub org which represents my organization. I would like to list all the teams in my GitHub org along with users in each team. This will assist me in creating namespaces and assigning roles using GitHub teams.

- I am a data scientist and part of the GitHub org which represents my organization. I would like to list all the repos in my GitHub org. This will assist me in making accurate decisions when creating namespaces and assigning roles using GitHub repos.

- I am a data scientist and part of the GitHub org which represents my organization. I would like to list all the teams and their associated roles for a repos in my GitHub org. This will assist me in making accurate decisions when creating namespaces and assigning privileges using GitHub repos.

- I am a data scientist and admin for a namespace on my k8s cluster. Members often join and leave teams or team and their roles are changed in my GitHub org. I would like to synchronize the users and their roles based on the original method which was used to inherit permissions from GitHub (using teams or repo) when creating the namespace.

- I am a data scientist who is a valid user on the k8s cluster. I would like to create a namespace and provide teams and their roles directly using a YAML file. Modeled on GitHub roles, this YAML based approach also supports Admin, Write and roles
  - Entries with invalid users and roles in the YAML file will be ignored.
  - Namespace must not exist on the cluster

### Proposed Changes
- Modify bootstrapper so that it creates a namespace with the annotation `managed-namespace`

#### New Components
- Role definitions:

```
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: member.kubeflow.org
spec:
  group: kubeflow.org
  version: v1
  scope: Namespaced
  names:
    plural: members
    singular: members
    kind: Member
    shortNames: ["mbr"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: kubeflow:admin,
rules:
- apiGroups:
  - *
  resources:
  - *
  verbs:
  - *
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: kubeflow:write,
rules:
- apiGroups:
  - *
  resources:
  - configmaps,
    pods,
    services,
    endpoints,
    persistentvolumeclaims,
    events
  verbs:
  - create, get, update, list, watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: kubeflow:read,
rules:
- apiGroups:
  - *
  resources:
  - configmaps,
    pods,
    services,
    endpoints,
    persistentvolumeclaims,
    events
  verbs:
  - get,
    watch,
    list
```
- A github controller that adds new behavior to Namespace related to RBAC users
- The GitHub controller is a DecoratorController that also attaches a custom resource to the Namespace: Organization.

```
apiVersion: metacontroller.k8s.io/v1alpha1
kind: DecoratorController
metadata:
  name: github-controller
spec:
  resources:
  - apiVersion: v1
    resource: namespaces
    annotationSelector:
      matchExpressions:
      - {key: managed-namespace, operator: Exists}
  attachments:
  - apiVersion: kubeflow.org/v1
    resource: Organization
  hooks:
    sync:
      webhook:
        url: http://github-controller.metacontroller/github
---
```

#### Changes to Existing Components
 - None

## Roadmap

### Phase 1
 - GitHub controller
 - No support for synchronizing roles

### Phase 2
- Support for GitHub role synchronization
- Ability to audit roles and flag anomalies
- Logging support

### Phase 3
- Updates per feedback from community

## Opens

## Challenges
- Accurate roles for admin, write and read access will require deeper discussions with data scientist. The proposal depends on these role definitions.

- Some of the GitHub API are in early stages (ex: Teams) and may be unstable.

## Limitations
- The proposal is only a step in addressing security challenges in k8s. Additional work is required to address other aspects like service accounts, PV to PVC bindings, pod security policy configuration etc.

## Alternatives

- Design choice - use a client centric approach to solve the problem where a client side tool obtains the users and roles from GitHub and generates the YAML file with rolebindings. This eliminates the need to develop the controller.

- Manually configure the rolebinding - this is error prone and requires k8s expertise.


## References

https://github.com/liggitt/audit2rbac

https://developer.github.com/v3/repos/

https://github.com/NervanaSystems/dls-features/issues/58
