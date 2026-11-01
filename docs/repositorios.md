# Repositorios · biblioteca

Ocho repositorios, uno por pieza, igual que el grupo de VidalStore. Estos son los de **partida** de L9 (fin de L8,
ya separado en ocho): haces Fork de los ocho y los clonas **sin el sufijo `-l9`**, uno al lado del otro, en la misma
carpeta. El `compose.yml` de esta carpeta busca a sus hermanos con `../biblioteca-<pieza>`.

| Pieza | Repositorio | Carpeta |
|---|---|---|
| frontend | https://github.com/Umbingelelo/biblioteca-web-l9 | raíz |
| gateway | https://github.com/Umbingelelo/biblioteca-gateway-l9 | raíz |
| bff | https://github.com/Umbingelelo/biblioteca-bff-l9 | raíz |
| libros | https://github.com/Umbingelelo/biblioteca-libros-l9 | raíz |
| prestamos | https://github.com/Umbingelelo/biblioteca-prestamos-l9 | raíz |
| eventos | https://github.com/Umbingelelo/biblioteca-eventos-l9 | raíz |
| admin | https://github.com/Umbingelelo/biblioteca-admin-l9 | raíz |
| plataforma | https://github.com/Umbingelelo/biblioteca-plataforma-l9 | raíz |

Cada repositorio trae su `Dockerfile` y, si lee variables, su `.env.example` con los nombres y sin los valores. Lo
único compartido es esta carpeta: `compose.yml`, `.env.example`, `docs/` y `rabbitmq/`.
