# MaaS console templates for OpenShift AI

Two OpenShift templates that publish a deployed model through Models-as-a-Service
(MaaS) and add priced subscription tiers to it. They are built to be filled in from
the OpenShift console form (**Instantiate Template**), so a platform team can
onboard models and tiers without writing YAML.

| Template | Creates | Run |
|---|---|---|
| **MaaS 1 - Publish a model** (`maas-publish-model`) | `MaaSModelRef` | Once per model. Skip it if the model was published as a MaaS endpoint from the OpenShift AI dashboard. |
| **MaaS 2 - Model subscription and pricing** (`maas-model-subscription-pricing`) | `MaaSSubscription`, `MaaSAuthPolicy`, `PrometheusRule` | Once per tier per model (Premium, Basic, …) |

The subscription and the auth policy always come together. Without the policy, the
gateway refuses the group (403). Without the subscription, the group gets through but
has no token budget (429). The PrometheusRule records the tier's price per million
tokens so the MaaS cost dashboard can turn token counts into cost.

## Repository layout

| Path | Purpose |
|---|---|
| `templates/maas-publish-model-template.yaml` | Template 1 |
| `templates/maas-model-subscription-pricing-template.yaml` | Template 2 |
| `rbac/maas-template-instance-governance-rbac.yaml` | One-time cluster setup required for console use (see below) |
| `examples/*.params.env` | Example parameter values, for the CLI path |
| `docs/console-guide.html` | Step-by-step customer guide for the console path, with troubleshooting |

## One-time setup (cluster admin)

```bash
# 1. Let the template controllers create and delete MaaS subscriptions and auth policies
oc apply -f rbac/maas-template-instance-governance-rbac.yaml

# 2. Add both templates to the tenant project, so they appear in its catalog
oc apply -n models-as-a-service -f templates/
```

Both steps also work from the console: **+ (Import YAML)**, paste, **Create**.

Why step 1 exists: when a template is instantiated from the console, the objects are
created by `openshift-infra/template-instance-controller` and removed (when the
TemplateInstance is deleted) by `openshift-infra/template-instance-finalizer-controller`.
Their built-in `admin` role covers `MaaSModelRef` and `monitoring.rhobs` PrometheusRules
but not `MaaSSubscription` or `MaaSAuthPolicy`. Without the binding:

- instantiating template 2 fails with
  `maassubscriptions.maas.opendatahub.io is forbidden: User "system:serviceaccount:openshift-infra:template-instance-controller" cannot create ...`
- deleting a TemplateInstance hangs, and the tier's subscription, auth policy and
  API keys stay in place.

The role is bound only to those two service accounts. Users gain no new rights.

## Use from the console

1. Select project `models-as-a-service`, open the software catalog, filter by type
   **Template**, and pick a template. Or open the form directly:
   ```
   https://<console-host>/catalog/instantiate-template?template=maas-publish-model&template-ns=models-as-a-service&preselected-ns=models-as-a-service
   https://<console-host>/catalog/instantiate-template?template=maas-model-subscription-pricing&template-ns=models-as-a-service&preselected-ns=models-as-a-service
   ```
2. Fill in the form and click **Create**. Every field has a description in the form.
3. To remove a tier, delete its TemplateInstance (Home → Search → TemplateInstance).
   That removes its subscription, auth policy and price rule.

`docs/console-guide.html` has the full walkthrough, field reference and troubleshooting.

## Use from the CLI

```bash
oc process --local -f templates/maas-publish-model-template.yaml \
  --param-file examples/publish-model.params.env | oc apply -f -

oc process --local -f templates/maas-model-subscription-pricing-template.yaml \
  --param-file examples/subscription-pricing.params.env | oc apply -f -
```

`--local` renders on your machine and `oc apply` runs as you, so the RBAC from the
setup step is not needed on this path. Unlike the console, `oc apply` also updates a
MaaSModelRef that already exists.

## Things to know

- **Token limits are per user.** The generated TokenRateLimitPolicy counts by
  `auth.identity.userid`. A group of 50 on a 20,000 tokens/minute tier can use up to
  50 × 20,000 tokens per minute in total. Prompt and completion tokens count together.
- **Give every tier a different priority.** Equal priorities raise
  `SpecPriorityDuplicate` on the subscriptions, and a new API key could bind to either.
- **Price matching.** `METRICS_MODEL_NAME` must equal the `model` field the model
  returns in its responses; that is how the price joins to usage. For an on-cluster
  model it is normally the LLMInferenceService's `spec.model.name`.
- **One currency everywhere.** The shipped cost dashboard joins prices with
  `group_left()` and does not carry the `currency` label, so mixed units are added
  together silently. Cents (`USc`) read better at pilot volumes.
- **Reusing a name fails.** The console creates objects and never updates them:
  `already exists` on template 1 means the model is already published; on template 2,
  pick a different subscription name.
- **Deleting a MaaS 1 instance unpublishes the model** for every tier. Remove the
  model's tiers first.

## Validated on

OpenShift 4.21.33, Red Hat OpenShift AI 3.5 (MaaS `maas.opendatahub.io/v1alpha1`),
Cluster Observability Operator monitoring stack in `redhat-ods-monitoring`.

Tested by submitting the same parameter Secret and TemplateInstance the console
creates: one model published with template 1, Premium (20,000 tokens/min) and Basic
(100 tokens/min) tiers added with template 2. Both tiers served requests, Basic
returned 429 after about 118 tokens in a minute, cost per tier was priced correctly
(118 × 50/1M = 0.0059 USc; 172 × 200/1M = 0.0344 USc), and deleting a tier's
instance removed its three objects and invalidated its API keys (403).
