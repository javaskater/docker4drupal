## Context:
* I am working on a Windows Laptop but working under Linux thanks to
  * [WSL2](https://learn.microsoft.com/en-us/windows/wsl/about)
  * [Docker for Windows](https://www.docker.com/products/personal/) running on top of the Ubuntu distribution brought by WSL2

## Where the docker configuration comes from
* I downloaded the official [Docker4Drupal GitHubProject](https://github.com/wodby/docker4drupal)
  * I saved the README at that time (Drupal version 9.4.8) under [Original README](README.origin.md)
* I reworked some _docker-compose_ or _docker-compose.overrides_ files
  * I explains all that in my documentation
## Where to find my documentation ?
* All I make to do it working can be found on this project at __MyDocs__ folder
* Actually there are the following files:
  *  [MyDocs](MyDocs/maraidb.md) for storing permanently the content of the Database
  *  [MyDocs](MyDocs/DockerOnWinows.md) for starting a Drupal local Web Site using Traefik on Docker for Windows running above WSL2
