###### tags: `UnsafeFailForward`

# Opting in to UnsafeFailForward upgrades on OpenShift Container Platform 4.11

`UnsafeFailForward` is a Technology Preview feature only. Technology Preview features are not supported with Red Hat production service level agreements (SLAs) and might not be functionally complete. Red Hat does not recommend using them in production.

For more information about the support scope of Red Hat Technology Preview features, see https://access.redhat.com/support/offerings/techpreview/.

**WARNING**
`UnsafeFailForward` is a feature that allows for the automatic upgrade of failed installs and upgrades. This feature is not recommended, as enabling `UnsafeFailForward` does not guarantee sane upgrade paths and might cause unrecoverable failures resulting in data loss.

Only use this feature if you:

- Know every Operator installed in the namespace.
- Have deep knowledge regarding the upgrade paths for each Operator in the namespace.
- Have control over the contents of the catalogs providing the Operators in the namespace.
- Do not want to manually upgrade your failed Operators.

## Understanding `UnsafeFailForward` upgrades

A failed installation or upgrade is typically caused by one of two scenarios:

- **A Failed CSV:** The Cluster Service Version [(CSV)](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) is in the `FAILED` phase.
- **A Failed InstallPlan:** Usually occurs because a resource listed in the [InstallPlan](https://olm.operatorframework.io/docs/concepts/crds/installplan/) fails to be created or updated. An InstallPlan may fail independently of its CSV and may fail to create the CSV.

By opting into `UnsafeFailForward` upgrades, Operator Lifecycle Manager (OLM) allows you to recover from failed installations and upgrades by:

- Allowing CSVs to move from the `FAILED` phase to the `REPLACING` phase.
- Allowing OLM to calculate new InstallPlans for a set of installables if:
     - The InstallPlan referenced by a Subscription is in the `FAILED` phase.
     - The CatalogSource has been updated to include a new upgrade for one or more CSVs in the namespace.

Before using `UnsafeFailForward` upgrades, it is important to understand how OLM and the Resolver calculate what needs to be installed. When determining what to install, OLM provides the Resolver with the set of CSVs in a namespace. The Resolver treats these CSVs differently depending on whether or not they are claimed by a Subscription, where the Subscription lists the CSV's name in its `.status.currentCSV` field.

If a CSV is claimed by a Subscription, the Resolver will allow it to be upgraded by a bundle that replaces it. If a CSV is not claimed by a Subscription, it is assumed that the user installed the CSV themselves, that OLM should not upgrade the CSV, and that it must appear in the next set of installables.

The OLM Resolver will not change unless new upgrades are declared in the `CatalogSource` fields. When an upgrade fails due to a CSV entering the `FAILED` phase, the CSV that has been replaced still exists and is not referenced by a Subscription. OLM sends both `Operator v1` and `Operator v2` to the Resolver which will normally be unsatisfiable because `Operator v1` is marked as required and `Operator v2` cannot be upgraded further since it provides the same APIs as `Operator v1`. To support failing forward, OLM needs to omit CSVs in the `REPLACING` phase from the set of arguments sent to the Resolver when `UnsafeFailForward` upgrades are enabled.

When you opt into `UnsafeFailForward` upgrades, OLM does not include CSVs in the `REPLACING` phase in the set of arguments sent to the Resolver if the final CSV in the upgrade chain is in the `FAILED` phase and all other CSVs are in the `REPLACING` phase. OLM allows CSVs to move from the `FAILED` phase to the `REPLACING` phase.

These changes allow OLM to recover from any number of failed CSVs in an upgrade path, replacing the latest `FAILED` CSV with the next available upgrade.

## Using UnsafeFailForward Upgrades

`UnsafeFailForward` is restricted to namespaces. The namespace-scoped `OperatorGroup` resource contains the `upgradesStrategy` field:

~~~
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: foo
  namespace: bar
spec:
  upgradeStrategy:
     # Possible values include `Default` or `TechPreviewUnsafeFailForward`.
    name: TechPreviewUnsafeFailForward
~~~

Setting the `upgradeStrategy` to `TechPreviewUnsafeFailForward` enables `UnsafeFailForward`. If the `upgradeStrategy` is unset or set to `Default`, OLM maintains its existing behavior.


### Upgrading when an InstallPlan fails

The following steps represent a typical troubleshooting workflow.

- `Operator v1` is upgrading to `Operator v2`.
- The InstallPlan for `Operator v2` fails.
- `Operator v3` is added to the catalog and is defined as the upgrade for `Operator v1`.
- The user deletes the InstallPlan created for `Operator v2`.
- A new InstallPlan is generated for `Operator v3` and the upgrade succeeds.

With `UnsafeFailForward` upgrades, OLM allows new InstallPlans to be generated if the failed InstallPlan is referenced by a Subscription in the namespace or the `CatalogSource` has been updated to include a new upgrade for the set of arguments.

The fourth step from the previous workflow would be removed, meaning that the you need to update the `CatalogSource` and wait for the cluster to move past the failed install. With catalog polling enabled, you can even skip updating the `CatalogSource` directly by pushing a new catalog image to the same tag.

**NOTE:** If a bundle is known to fail, it can be skipped in the upgrade graph using the [skips](https://olm.operatorframework.io/docs/concepts/olm-architecture/operator-catalog/creating-an-update-graph/#skips) or [skipRange](https://olm.operatorframework.io/docs/concepts/olm-architecture/operator-catalog/creating-an-update-graph/#skiprange) feature.

![!alt](https://access.redhat.com/sites/default/files/attachments/failedinstallplan.jpg)

### Upgrading one or more CSVs fail

The following steps represent a typical troubleshooting workflow.

- `Operator v1` is being upgraded to `Operator v2`.
- The CSV for `Operator v2` enters the FAILED phase.
- `Operator v3` is added to the catalog which replaces or skips `Operator v2`.
- The Resolver cannot upgrade `Operator v2` while including `Operator v1` in the solution set. The upgrade is blocked.
- The user manually deletes the existing CSVs, and a new InstallPlan is generated and approved.
- `Operator v3` is installed and the upgrade succeeds.

![!alt](https://access.redhat.com/sites/default/files/attachments/failedcsvs.jpg)