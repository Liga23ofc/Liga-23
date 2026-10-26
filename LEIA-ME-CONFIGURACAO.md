# Liga 23 — atualização da área de clubes

## O que foi adicionado
- Aba “Área dos clubes” integrada ao site existente.
- Formulário de login/criação de conta (Firebase Authentication).
- Cadastro de clube, sigla, responsável, contato, escudo e elenco (mínimo de 5 jogadores).
- Upload do comprovante Pix e do escudo para Firebase Storage.
- Painel do representante para consultar o estado da inscrição.
- Painel administrativo simples para aprovar/recusar inscrições.
- Mantidas as seções anteriores do site, incluindo classificação, jogos, Copa, regulamento, WhatsApp e Pix.

## IMPORTANTE: ainda não está pronto para receber inscrições reais
O repositório original é um site estático e não continha backend. A interface foi preparada para Firebase, mas **você precisa configurar o projeto Firebase antes de publicar esta versão**. Até lá, login e envio de cadastros não funcionam; o site informa isso ao visitante.

## Configuração do Firebase
1. Crie um projeto em https://console.firebase.google.com/.
2. Adicione um app Web e copie a configuração do Firebase.
3. No `index.html`, substitua os valores de `firebaseConfig` (`COLOQUE_SUA_API_KEY`, `SEU_PROJETO`, `SEU_SENDER_ID`, `SEU_APP_ID`) pelos valores do seu app.
4. Troque `SEU_EMAIL_ADMIN@EXEMPLO.COM` pelo e-mail que você usará como administrador. Use uma conta de e-mail controlada por você.
5. No Firebase Authentication, ative o provedor “E-mail/senha”.
6. Crie o Firestore Database e o Storage.
7. Publique regras seguras (exemplo abaixo) antes de aceitar dados reais. Faça teste com uma conta comum e uma conta de administrador.
8. Publique os arquivos no repositório GitHub Pages.

## Regras iniciais sugeridas para Firestore
Atenção: regras de segurança são essenciais. O exemplo abaixo deixa o proprietário ler seus registros e permite ao administrador revisar. Como o status de aprovação e o cadastro são escritos no mesmo documento, para segurança máxima recomenda-se separar os dados editáveis do clube dos campos administrativos ou implementar uma Cloud Function para validar a criação. Não habilite permissões públicas.

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() { return request.auth != null; }
    function isAdmin() {
      return signedIn() && request.auth.token.email in ['SEU_EMAIL_ADMIN@EXEMPLO.COM'];
    }
    match /clubApplications/{applicationId} {
      allow create: if signedIn()
        && request.resource.data.ownerUid == request.auth.uid
        && request.resource.data.ownerEmail == request.auth.token.email
        && request.resource.data.applicationStatus == 'pending'
        && request.resource.data.paymentStatus == 'pending';
      allow read: if signedIn() && (resource.data.ownerUid == request.auth.uid || isAdmin());
      allow update: if isAdmin();
      allow delete: if isAdmin();
    }
  }
}
```

## Regras iniciais sugeridas para Storage
Ajuste o e-mail administrativo para o seu. Limite de tamanho e tipos de arquivo devem ser validados nas regras do Storage também.

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    function signedIn() { return request.auth != null; }
    function isAdmin() {
      return signedIn() && request.auth.token.email == 'SEU_EMAIL_ADMIN@EXEMPLO.COM';
    }
    match /club-uploads/{uid}/{fileName} {
      allow read: if signedIn() && (request.auth.uid == uid || isAdmin());
      allow write: if signedIn() && request.auth.uid == uid
        && request.resource.size < 2 * 1024 * 1024
        && request.resource.contentType.matches('image/(png|jpeg|webp)');
    }
    match /payment-proofs/{uid}/{fileName} {
      allow read: if signedIn() && (request.auth.uid == uid || isAdmin());
      allow write: if signedIn() && request.auth.uid == uid
        && request.resource.size < 5 * 1024 * 1024
        && (request.resource.contentType.matches('image/(png|jpeg|webp)') || request.resource.contentType == 'application/pdf');
    }
  }
}
```

## Observações
- O fluxo inicial usa Pix manual: o clube paga R$ 5,00 e envia comprovante; o administrador confere antes de aprovar.
- A configuração Pix existente foi preservada do arquivo recebido. Confirme valor, chave, titular e QR Code com um pagamento de teste antes de divulgar.
- Os links dos arquivos enviados são protegidos pelas regras de Storage; não compartilhe URLs publicamente.
- Para publicar dados de clubes aprovados automaticamente na página pública, ou gerenciar tabela/resultados no painel, ainda será preciso implementar essa sincronização. Esta versão não afirma fazer isso automaticamente.
- Recomenda-se ativar verificação de e-mail, proteção contra abuso e uma política de privacidade antes de abrir inscrições.
