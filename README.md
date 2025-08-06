# RaspPi_WebHost

### Deployment Workflow
- Source code cloned via Git into Raspberry Pi
- Backend built and published with `dotnet publish`
- Angular frontend built on dev machine, transferred via SCP
- Nginx serves Angular, proxies API requests to ASP.NET Core
- systemd service ensures backend auto-starts on boot