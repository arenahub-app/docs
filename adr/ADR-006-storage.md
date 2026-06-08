# ADR-006 — Storage de Arquivos: Cloudflare R2

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

O sistema precisa armazenar:
- Fotos de perfil dos usuários
- Fotos dos grupos
- Comprovantes de pagamento (JPEG, PNG, PDF)

Requisitos:
- API S3-compatible para facilitar futura migração
- Custo baixo para MVP (sem egress fees)
- Suporte a upload direto do cliente (presigned URLs) para descarregar o backend
- Controle de acesso (arquivos não públicos por padrão)

## Decisão

**Serviço:** Cloudflare R2  
**Estratégia de upload:** Presigned URLs (cliente envia diretamente ao R2; backend assina a URL)

### Fluxo de Upload

```
1. Cliente solicita URL de upload ao backend
   POST /api/v1/storage/upload-url
   Body: { fileName, contentType, context }

2. Backend valida tipo e tamanho permitidos
   Gera presigned PUT URL para o R2 (validade: 5 min)
   Retorna: { uploadUrl, fileKey }

3. Cliente faz PUT direto no R2 com o arquivo
   R2 armazena o arquivo

4. Cliente notifica o backend que o upload foi concluído
   Backend registra a referência (fileKey) no banco de dados

5. Para visualização, backend gera presigned GET URL com validade curta
   (evita hotlinking e controla acesso por autenticação)
```

### Organização de Buckets

```
arenahub-prod/
├── profiles/           # Fotos de perfil
│   └── {userId}/photo.{ext}
├── groups/             # Fotos de grupo
│   └── {groupId}/cover.{ext}
└── payments/           # Comprovantes
    └── {groupId}/{paymentId}/{timestamp}.{ext}
```

### Validações no Backend

- Tipo MIME: `image/jpeg`, `image/png`, `application/pdf`
- Tamanho máximo: 10MB (comprovantes), 5MB (fotos)
- O backend valida o Content-Type antes de emitir a presigned URL

## Alternativas Consideradas

### Opção A: AWS S3
- **Prós:** Mais maduro; documentação extensa; amplamente suportado
- **Contras:** Egress fees (cobrado por download); custo maior para MVP; requer conta AWS

### Opção B: Cloudflare R2 (escolhida)
- **Prós:** Zero egress fees; API S3-compatible; integrado ao Cloudflare CDN; custo muito baixo para MVP; fácil migração para S3 se necessário
- **Contras:** Menos maduro que S3; sem aceleração por regiões (único datacenter por bucket)

### Opção C: Armazenamento local (disco do servidor)
- **Prós:** Zero custo; simples de implementar no MVP
- **Contras:** Inviável para escala horizontal; sem redundância; não funciona com múltiplas instâncias

### Opção D: Supabase Storage
- **Prós:** Integrado com banco; UI administrativa incluída
- **Contras:** Lock-in no Supabase; não é a escolha de banco do projeto

## Consequências

**Positivas:**
- Zero egress fees = custo previsível para MVP
- API S3-compatible = migração para AWS S3 trivial no futuro
- Upload direto = backend não processa bytes de arquivo; menor uso de memória/CPU

**Negativas:**
- Presigned URLs requerem implementação cuidadosa de expiração e controle de acesso
- Limpeza de arquivos órfãos requer job periódico (arquivos enviados mas não confirmados no banco)

## Adapter Pattern

```java
// Porta (interface) no domínio/aplicação
public interface FileStoragePort {
    PresignedUploadUrl generateUploadUrl(String fileKey, String contentType, Duration validity);
    PresignedDownloadUrl generateDownloadUrl(String fileKey, Duration validity);
    void delete(String fileKey);
}

// Implementação no infrastructure (adaptador)
public class R2StorageAdapter implements FileStoragePort { ... }
```

Isso permite substituir o R2 por qualquer storage S3-compatible sem alterar código de aplicação.
