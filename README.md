# Helm Paketmanagement

## Agenda 

  1. Kubernetes
     * [Architektur Kubernetes](/kubernetes/architecture.md)
     * [Aufbau von Browser zu Applikation - Schaubild](kubernetes/konzepte/anatomie-einer-applikation.md)

  1. Helm Einfuehrung 
     * [Was ist helm ?](einfuehrung/was-ist-helm.md)
     * [Was kann helm ?](einfuehrung/was-kann-helm.md)
     * [Was ist in helm ein chart?](einfuehrung/helm-chart.md)
     * [Warum Helm in Kubernetes verwenden ?](einfuehrung/warum-helm-verwenden.md)
     * [Überblick über den Ablauf bei der Nutzung von helm (Kommando: install)](einfuehrung/ablauf-helm-install.md)
     * [Braucht helm das Programm kubectl ?](einfuehrung/braucht-helm-kubectl.md)
       
  1. Helm Installation und Konfiguration (inkl. kubectl) 
     * [Installation von kubectl unter Linux](kubectl/installation/linux.md)
     * [Konfiguration von kubectl mit namespaces](kubectl/kubectl-einrichten.md)
     * [Installation von helm unter Linux](helm/installation/linux.md)
     * [Installation bash completion](helm/installation/bash-completion.md)

  1. Helm Grundlagen
     * [TopLevel Objekte](/helm/grundlagen/toplevel-objekte.md)
      
  1. Helm - Spickzettel
     * [Wichtig: Helm Spickzettel](helm/spickzettel.md)

  1. Arbeiten mit helm - charts
     * [Installation, Upgrade, Uninstall helm-Chart exercise](helm/commands/install.md)
     * [Nur fertiges manifest ausgeben ohne Installation](helm/commands/template.md)
     * [Informationen aus nicht installierten Helm-Charts bekommen](helm/commands/show.md)
     * [Chart runterladen und evtl. entpacken und bestimmte Version](/helm/commands/pull.md)
     * [Aufräumen von CRD's nach dem Deinstallieren](helm/exercises/08-exercise-crd-certmanager.md)

  1. Helm Charts entwickeln
     * [eigenes helm chart erstellen (Gruppe)](/helm/exercises/04a-create-chart-my-app-gruppenarbeit.md)    

  1. Spezial: Umgang mit Einrückungen
     * [Whitespaces meistern mit "-"](basics/whitespace-management.md)
     * [Exercise Whitespaces](/helm/templates/spaces.md)

  1. Type - Conversions
     * [Exercise toYaml](helm/exercises/01-typeConversionsToYaml.md)
    
  1. Flow Control
     * [if](/helm/templates/flow-control/02-only-if.md)
     * [with](/helm/templates/flow-control/02-with.md)
     * [range](/helm/templates/flow-control/03-range.md)

  1. Named Templates
     *  [named template](helm/exercises/10-named-template.md)
     *  [named template with dict](/helm/exercises/11-named-template-with-dict.md)
    
  1. Helm mit gitlab ci/cd
     * [Helm mit gitlab ci/cd ausrollen](helm/gitlab-ci-cd/example-helm-kubernetes.md)

  1. Metrics - Server
     * [Metrics - Server mit helm installieren und verwenden](/helm/metrics-server.md)
    
  1. helm - Dokumentation
     * [Helm Documentation](https://helm.sh/docs/)

## Backlog 

  1. Grundlagen
     * [Feature / No-Features von Helm](/helm/grundlagen/features-no-features.md)
   
  1. Tipps & Tricks
     * [kubernetes manifests mit privatem Repo](helm/exercises/02-pod-from-private-repo.md)
     * [helm chart mit images auf privatem Repo](helm/exercises/03-helm-nginx-image-from-private-repo.md)

  1. Helm-Befehle und -Funktionen
     * [Repo einrichten](/helm/commands/repo.md)
     * [Suche in Repo und Artifacts Hub](/helm/commands/search.md)
     * [Anzeigen von Informationen aus dem Chart von Online](/helm/commands/show.md)
     * [Upgrade und auftretende Probleme](/helm/commands/upgrade.md)

 1. Helm Repository
     * [Die wichtigsten Repo-Befehle](helm/commands/repo.md)

  1. Struktur von Helm - Charts
     * [Überblick](helm/structure/overview.md)

  1. Grundlagen Helm-Charts
     * [Testumgebung und Spaces (2 Themen)](/helm/templates/spaces.md)

  1. Erstellen von Helm-Charts
     * [Erstellen eines Guestbooks](helm/create-charts/guestbook/01-guestbook.md)
     * [Hooks für Guestbook erstellen](/helm/create-charts/guestbook/02-guestbook-verbessern.md)
     * [Dependencies/Abhängigkeiten herunterladen](helm/create-charts/download-dependencies.md)
     * [Einfaches Testen](helm/test/simple-test.md)
     * [Input Validierung innerhalb von templates](helm/input-validation/example.md)
     * [Advanced Testing mit chart-testing](helm/test/advanced-testing/advanced-testing-with-chart-testing.md)
     * [Chart auf github veröffentlichen](helm/create-charts/publish/publish-on-github.md)

  1. Sicherheit von helm-Chart
     * [Grundlagen / Best Practices](helm/security/best-practices.md)
     * [Security Encrypted Passwords in helm](/helm/security/secrets-password.md)

  1. Testing in Helm-Charts
     * [Testing in/von helm - charts](/helm/test/helm-test.md)

  1. Durchführung von Upgrades und Rollbacks von Anwendungen

  1. Helm in Continuous Integration / Continuous Deployment (CI/CD) Pipelines

  1. Tipps & Tricks
     * [Set namespace in config of kubectl](/kubectl/set-namespace-in-config.md)
     * [Create Ingress Redirect](/helm/create-charts/example-ingress.md)
     * [Helm Charts - Development - Best practices](https://helm.sh/docs/howto/charts_tips_and_tricks/)

  1. Integration mit anderen Tools
     * [yamllint für Syntaxcheck von yaml - Dateien](helm/tools/yamllint.md)

  1. Troubleshooting und Debugging
     * [helm template --validate - gegen api-server testen](helm/test/helm-template-validate.md)
