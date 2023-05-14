# k8s quick wins

This repo is for a talk I will hold at a local meetup.
It showcases a few basics security quick wins that is easy to apply to most orgs.

This repo will go through

- create a basic deployment
- DON't RUN YOUR CONTAINER AS ROOT!
- add security context
  - disable root account
  - disable write to disk
  - etc
- Requests/limits
  - default limits
- Disable service account token creation
- Talk about yaml linting tools that can help with best practices
- OPA/kyverno
  - mutations
- AntiAffinity
- PDB
  - PDB and HPA don't know about each other
- image scanning
- network policy
- service mesh
  - istio
  - linkerd
  - cilium
  - istio ambient mesh
