# Machine Config

## MC for Chrony config

- **EXAMPLE chronyc config**
```
server time1.google.com iburst
server 216.239.35.0 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

- **gzip and base64**
```
cat <chrony.conf> | gzip | base64 -w0
``
replace it in mc-chrony.yaml`
