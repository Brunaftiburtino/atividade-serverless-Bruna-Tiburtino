# Roteiro do vídeo (máx. 3 min, gravação de tela)

1. (0:00) Mostrar `sam --version`, `docker ps` e a árvore do projeto. Falar o nome.
2. (0:20) `sam build` e `sam local invoke -e events/pedido.json`, duas vezes. Apontar o Duration.
3. (1:00) `sam local start-api` e três `curl`. Apontar `invocacaoNesteAmbiente` = 1,1,1.
4. (1:30) Reiniciar com `--warm-containers LAZY` e repetir: 1,2,3.
5. (2:00) `curl` com `"demorar":true` e mostrar o erro de timeout (Timeout 3).
6. (2:30) Opcional: Timeout 10, `sam build`, repetir e mostrar o 201.
