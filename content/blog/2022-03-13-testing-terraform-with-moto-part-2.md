---
title: "Testing terraform with moto, part 2"
date: 2022-03-13T17:00:00
draft: true
---

I had previously (exported and converted the terraform output of builder to HCL)[https://github.com/elifesciences/kubernetes-cluster-provisioning], and decided to build upon that work here mostly because of fewer moving parts involved.


I had an (in-progress branch)[https://github.com/elifesciences/kubernetes-cluster-provisioning/tree/refactor-into-a-module] that extracted the commonality in the generated terraform config into a terraform module, providing a better basis on which to create and inject a provider pointing to Moto.
