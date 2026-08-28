# Task Participation Summary

## What I worked on

After discussions with my mentor Lucas, we decided to work on improving Magma deployment especially for local development. Currently Magma deployment is a little complicated and there are a lot of issues you will encounter during it. My objective was working on the Minikube deployment and after validation we will integrate it in magma-deployer.

## Contributions

- [PR #16032](https://github.com/magma/magma/pull/16032): Fixed deprecated Kubernetes API versions (`policy/v1beta1` to `policy/v1`) across 27 YAML files in both the orc8r and lte-orc8r Helm charts. Without this fix, `helm install` fails on Kubernetes 1.25+.
- [PR #16034](https://github.com/magma/magma/pull/16034), [PR #80](https://github.com/magma/magma-documentation/pull/80): Updated the Minikube deployment guide: fixed the Kubernetes version, added the fluentd gem-pin workaround, the cert ownership fix, and a new section for installing the LTE chart.
- Wrote deployment guides for the [AGW](guides/agw-deployment.md), [orc8r on Minikube](guides/orc8r-deployment-guide.md), and the [LTE module installation](guides/install-lte-module.md), documenting every issue and fix at the exact step where it occurs.
- Both PRs were validated on a clean AWS instance (t3.xlarge, Ubuntu 22.04) to confirm reproducibility before submitting.

## What I learned

For me this was a very useful mentorship either on the technical side or the communication side. For technical stuff I learned about Kubernetes, Minikube, Helm charts, 4G/LTE, and telecommunication in general, which connects directly to my studies in CS & telecommunication, I also learned how to use srsRAN 4G to simulate a full LTE attach using ZeroMQ virtual radios instead of real radio hardware, beyond that I got experience debugging across multiple layers (VirtualBox, WSL, Docker, Kubernetes, kernel modules), reading C++ source code to trace issues that config changes couldn't fix, and capturing network traffic with Wireshark and tcpdump for protocol analysis.

I also encountered a lot of subjects when working on Linux that are related to LPIC 1 that I'm currently preparing for, so it was beneficial in both sides.

## What was challenging

The challenge was understanding the roles of different Magma components. Magma is a big project with a lot of moving parts, so my challenge was trying to understand the role of each of them and how they communicate.