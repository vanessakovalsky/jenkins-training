# Jenkins training

Ce dépôt accompagne mes différentes sessions de formation autour de Jenkins


## Tips

### Erreur docker : ERROR: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock:

--> Solution : 

```
docker compose exec -u root -it jenkins bash
chmod 666 /var/run/docker.sock
```

/!\ Cette solution n'est pas sécurisé pour des contextes de production

