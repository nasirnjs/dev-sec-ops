

`docker run -v $(pwd):/tmp hcr.io/gitleaks/gitleaks:latest detect -r /tmp/gitleaks-report.json /tmp`

It will scan your current directory because of the -v $(pwd):/tmp volume mapping.

How It Works:
- $(pwd): This dynamically inserts the full path of your current working directory on the host. Example: If you're in /home/user/my-project, it mounts that folder to /tmp inside the container.
- Inside the container, you tell gitleaks to scan /tmp, which is mapped to your current directory on the host.
- -r /tmp/gitleaks-report.json specifies where the report will be saved.