# Code4ro k8s manifests

[![License: MPL 2.0][ico-license]][link-license]

This repository contains the k8s manifest for all the applications in the Code4ro platform.
The k8s cluster is using [ArgoCD](https://cd.code4.ro) to automatically deploy manifests when a new change is detected.

The flow is as follow:

1. User adds a new tag in application git repository
2. CI kicks in in that repository and builds the new image
3. The image is pushed to docker hub with that specific tag
4. A new commit is made by the CI/user on this repo in which we change the image version in the manifest (wait for the GHA that pushes the image to DockerHub to end successfuly)
5. ArgoCD will detect the change in this repo and apply the manifests to the k8s cluster

ArgoCD projects:

- **_infra_**: knative, cert-manager, sealed-secrets and argocd. `infra/argo-apps-infra.yaml` is the root ArgoCD Application for `infra/argo-apps`. `infra/argo-apps` store the ArgoCD Applications.
- **_default_**: all applications manifests. `apps/argo-apps-default.yaml` is the root ArgoCD Application for `infra/argo-apps`. `apps/argo-apps` store the ArgoCD Applications.

## Contributing

This project is built by amazing volunteers and you can be one of them! Here's a list of ways in [which you can contribute to this project][link-contributing]. If you want to make any change to this repository, please **make a fork first**.

## Feedback

- Request a new feature on GitHub.
- Vote for popular feature requests.
- File a bug in GitHub Issues.
- Email us with other feedback contact@code4.ro

## License

This project is licensed under the MPL 2.0 License - see the [LICENSE](LICENSE) file for details

## About Code for Romania

Started in 2016, Code for Romania is a civic tech NGO, official member of the Code for All network. We have a community of around 2.000 volunteers (developers, ux/ui, communications, data scientists, graphic designers, devops, it security and more) who work pro-bono for developing digital solutions to solve social problems. #techforsocialgood. If you want to learn more details about our projects [visit our site][link-code4] or if you want to talk to one of our staff members, please e-mail us at contact@code4.ro.

Last, but not least, we rely on donations to ensure the infrastructure, logistics and management of our community that is widely spread across 11 timezones, coding for social change to make Romania and the world a better place. If you want to support us, [you can do it here][link-donate].

[ico-contributors]: https://img.shields.io/github/contributors/code4romania/standard-repo-template.svg?style=for-the-badge
[ico-last-commit]: https://img.shields.io/github/last-commit/code4romania/standard-repo-template.svg?style=for-the-badge
[ico-license]: https://img.shields.io/badge/license-MPL%202.0-brightgreen.svg?style=for-the-badge
[link-contributors]: https://github.com/code4romania/standard-repo-template/graphs/contributors
[link-last-commit]: https://github.com/code4romania/standard-repo-template/commits/main
[link-license]: https://opensource.org/licenses/MPL-2.0
[link-contributing]: https://github.com/code4romania/.github/blob/main/CONTRIBUTING.md
[link-production]: insert_link_here
[link-staging]: insert_link_here
[link-code4]: https://www.code4.ro/en/
[link-donate]: https://code4.ro/en/donate/
