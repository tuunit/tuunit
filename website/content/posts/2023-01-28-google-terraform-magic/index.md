---
date: 2023-01-28T21:12:54+01:00
title: How Google applies "magic" to Terraform
---

I’ve always been curious about how big orgs ship features at scale. How they move ideas from alpha to beta to GA without breaking the minds of users and workflows in production. And how they run open source in public while wiring it to internal release trains. Have you ever wondered how Google keeps its Terraform providers in sync with fast-moving GCP features?

This is where Magic Modules comes in for building the official Terraform provider, validator and beta features provider from a single source through an interesting combination of Go and Embedded Ruby (ERB) templating.

![Magic Modules Logo](magic-modules.svg)

https://github.com/GoogleCloudPlatform/magic-modules

More about the details will follow below but first why did I need to investigate how Google manages its providers and the code generation? Because even though the Google Cloud is always at the cutting edge of Kubernetes topic. After all they created Kubernetes and still heavily govern and manage it. Even they don't always ship all the necessary parts to make the feature fly. Let me tell you the history:

In a project for a large customer of mine, we decided to go with the Gateway API to future proof the cluster for the eventual deprecation of Ingress. Yes we know it will be years but nevertheless, after all Google introduced the Gateway API to GKE in beta on [30th of April 2021](https://cloud.google.com/blog/products/containers-kubernetes/new-gke-gateway-controller-implements-kubernetes-gateway-api). It then became GA on [11st of November 2022](https://cloud.google.com/blog/products/containers-kubernetes/google-kubernetes-engine-gateway-controller-is-now-ga). 

But by late November of 2022 we noticed that the Google Terraform providers both beta and GA still lacked the necessary Gateway API config option to install the CRDs and controller. So at first we initialized our clusters through Terraform and then manually had to intervene to activate the Gateway API via gcloud cli before we could continue with the internals of GKE.

But wait a minute I thought to myself. The gcloud cli must be using the google go sdk for configuring GKE. So I started looking into it and after a while I found the [internals of the code](https://pkg.go.dev/google.golang.org/api/container/v1#GatewayAPIConfig).

Now how does this relate to the previously mentioned "magic". Most all Terraform providers are written in go and usually use the same SDKs as all other tooling like the Hyperscalers own cli tools. So I just had to figure out where to implement this feature into the Terraform providers of Google.

So how does the magic actual work under the hood?

As mentioned before Google uses a single source of truth for generating its Terraform providers in form of the so called [Magic Modules repository](https://github.com/GoogleCloudPlatform/magic-modules). Think of it as a spec plus templates that generate provider code. As mentioned previously this is done through utilizing the ERB (Embedded Ruby) templating processor with and for Go code. From one source, a mono repository a bot (pipeline) emits automated PRs with the dedicated Go code for the following repositories:
- https://github.com/hashicorp/terraform-provider-google
- https://github.com/hashicorp/terraform-provider-google-beta
- https://github.com/GoogleCloudPlatform/terraform-validator

Why this model works:
- Single source of truth for schema, docs, diffs, tests.
- Consistent behavior between beta and GA.
- Faster propagation as the PRs and merges to the upstream provider repositories are fully automated.

Why it can surprise you:
- You fix gaps in Magic Modules, not in the provider repos.
- Changes land with the next release cut, not instantly.
- Check Magic Modules first. If absent, add it there with acceptance tests.
- Expect it to show up in google-beta first, then google.
- Watch release notes to plan upgrades.

For Gateway API, we were a bit stumped but after looking around in the previously mentioned SDK docs. I decided to give it a shot and made it work in a couple hours in the evening. I added the new `gateway_api_config` in Magic Modules and opened the PR on the 26th of November, it was reviewed and merged on 13th of December and then the "magic" kicked in the [modular-magician](https://github.com/modular-magician) bot generated the provider changes, opened PRs in the necessary repositories and merged them. With the next release [v4.47.0](https://github.com/hashicorp/terraform-provider-google/releases/tag/v4.47.0) on the 21th of December 2022 we and others IaC were unblocked. 

If you are interested in my implementation checkout: https://github.com/GoogleCloudPlatform/magic-modules/pull/6875
