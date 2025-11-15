# Especificación de API

## API Pública para Integración con NFT Media Validator

### Principios de Diseño de la API

1. **RESTful**: Endpoints predecibles y semántica HTTP estándar
2. **Abierta**: Verificación gratuita, certificación con autenticación
3. **Versionada**: `/api/v1/` para compatibilidad futura
4. **Documentada**: OpenAPI 3.0 spec completa
5. **Rate-Limited**: Justos pero generosos límites
6. **Developer-Friendly**: SDKs en múltiples lenguajes

### Autenticación

#### Métodos Soportados

**1. API Key (Para Integraciones)**
```
Header: X-API-Key: <your_api_key>

Obtención:
- Dashboard → Settings → API Keys
- Crear nueva key con scopes específicos
- Rotación recomendada cada 90 días
```

**2. JWT (Para Usuarios Web)**
```
Header: Authorization: Bearer <jwt_token>

Obtención:
POST /api/v1/auth/login
Body: { wallet: "0x...", signature: "..." }
Response: { token: "eyJ...", expiresIn: 3600 }
```

**3. OAuth 2.0 (Para Apps de Terceros)**
```
Flujo estándar OAuth 2.0
Scopes:
- read:verify - Verificar medios
- write:certify - Certificar medios
- read:user - Leer perfil de usuario
- write:user - Modificar perfil
- read:analytics - Ver estadísticas
```

### Rate Limits

```
Tier            Verificación    Certificación   Burst
──────────────────────────────────────────────────────
Anonymous       100/min         10/min          150
Free            500/min         50/min          750
Pro             2000/min        500/min         3000
Enterprise      Unlimited       Unlimited       -
```

**Headers de Rate Limit**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1678901234
```

### Endpoints

#### 1. Certificación

**POST /api/v1/certify**

Certifica un nuevo medio y lo registra en blockchain.

```
Request:
Content-Type: multipart/form-data

Fields:
- media: File (required) - Imagen o video
- metadata: JSON (optional) - Metadata adicional
  {
    "title": "Protest in Main Square",
    "description": "...",
    "location": "40.7128,-74.0060",
    "capturedAt": "2025-01-15T14:30:00Z",
    "camera": "Canon EOS R5",
    "tags": ["protest", "news", "breaking"]
  }
- watermark: JSON (optional) - Configuración de marca visible
  {
    "enabled": true,
    "position": "bottom-right",
    "opacity": 0.7,
    "customText": "© John Doe 2025"
  }
- options: JSON (optional)
  {
    "skipAIDetection": false,
    "priority": "normal", // normal | high
    "preserveOriginal": true
  }

Response: 201 Created
{
  "success": true,
  "certificateId": "cert_a3f2b8d9",
  "hash": "a3f2b8d9e5c7f1a4b6d8e9f0a1b2c3d4...",
  "blockchainTx": "0x8d4e7f2a...",
  "ipfsCID": "QmX8f3g2h...",
  "authenticityScore": 95,
  "signedMediaUrl": "https://cdn.nftmv.io/signed/cert_a3f2b8d9.jpg",
  "certificateUrl": "https://nftmv.io/verify/cert_a3f2b8d9",
  "qrCodeUrl": "https://api.nftmv.io/v1/qr/cert_a3f2b8d9",
  "processingTime": 4.3,
  "metadata": {...},
  "createdAt": "2025-01-15T14:32:01Z"
}

Errors:
400 Bad Request - Invalid format, file too large, etc.
{
  "error": "INVALID_FORMAT",
  "message": "Only JPEG, PNG, MP4 are supported",
  "code": 40001
}

402 Payment Required - Insufficient credits
422 Unprocessable Entity - AI-generated detected, manipulated content
{
  "error": "AI_GENERATED_DETECTED",
  "message": "Content appears to be AI-generated",
  "authenticityScore": 35,
  "details": {...}
}

429 Too Many Requests - Rate limit exceeded
500 Internal Server Error - Server error
503 Service Unavailable - Blockchain temporarily unavailable
```

**POST /api/v1/certify/batch**

Certifica múltiples medios en una sola operación.

```
Request:
Content-Type: multipart/form-data

Fields:
- media[]: File[] (max 100)
- metadata[]: JSON[] (optional, matching array)

Response: 202 Accepted
{
  "batchId": "batch_x7y8z9",
  "totalItems": 50,
  "statusUrl": "/api/v1/batch/batch_x7y8z9",
  "estimatedCompletionTime": 120
}
```

**GET /api/v1/batch/:batchId**

Obtiene el estado de una certificación por lotes.

```
Response: 200 OK
{
  "batchId": "batch_x7y8z9",
  "status": "processing", // pending | processing | completed | failed
  "totalItems": 50,
  "completed": 32,
  "failed": 1,
  "pending": 17,
  "results": [
    {
      "index": 0,
      "status": "completed",
      "certificateId": "cert_...",
      ...
    },
    {
      "index": 5,
      "status": "failed",
      "error": "AI_GENERATED_DETECTED"
    }
  ]
}
```

#### 2. Verificación

**GET /api/v1/verify/quick/:hash**

Verificación rápida solo por hash.

```
Request:
GET /api/v1/verify/quick/a3f2b8d9e5c7f1a4b6d8e9f0a1b2c3d4

Response: 200 OK
{
  "exists": true,
  "certificateId": "cert_a3f2b8d9",
  "hash": "a3f2b8d9...",
  "creator": "0x742d...",
  "createdAt": "2025-01-15T14:32:01Z",
  "mediaType": "image/jpeg",
  "blockchainTx": "0x8d4e7f2a...",
  "revoked": false
}

Response: 404 Not Found
{
  "exists": false
}
```

**POST /api/v1/verify/standard**

Verificación estándar con análisis de integridad.

```
Request:
Content-Type: multipart/form-data

Fields:
- media: File (required)
OR
- url: String (URL to media)

Response: 200 OK
{
  "status": "verified", // verified | modified_minor | altered | not_found
  "exists": true,
  "certificate": {
    "certificateId": "cert_a3f2b8d9",
    "hash": "a3f2b8d9...",
    "creator": "0x742d...",
    "createdAt": "2025-01-15T14:32:01Z",
    "metadata": {...}
  },
  "integrity": {
    "hashMatch": true,
    "watermarkIntact": true,
    "invisibleSealIntact": true,
    "similarity": 100
  },
  "processingTime": 1.2
}

Response: Alterado
{
  "status": "altered",
  "exists": true,
  "certificate": {...},
  "integrity": {
    "hashMatch": false,
    "watermarkIntact": false,
    "invisibleSealIntact": true,
    "similarity": 76,
    "differences": {
      "pixelsChanged": 15324,
      "regionsAltered": 3,
      "suspiciousAreas": [...]
    }
  },
  "recommendations": [
    "Significant alterations detected",
    "Original available at: https://ipfs.io/ipfs/QmX8f3g2h..."
  ]
}
```

**POST /api/v1/verify/forensic**

Verificación forense profunda.

```
Request:
Content-Type: multipart/form-data

Fields:
- media: File (required)
- options: JSON (optional)
  {
    "generateReport": true,
    "includeDifferenceMap": true,
    "compareWithOriginal": true
  }

Response: 200 OK
{
  "status": "verified",
  "certificate": {...},
  "integrity": {...},
  "forensicAnalysis": {
    "authenticityScore": 95,
    "ela": {
      "score": 92,
      "anomalies": []
    },
    "cloneDetection": {
      "clonesFound": 0
    },
    "metadataForensics": {
      "exifConsistency": 98,
      "timestamps": {
        "created": "2025-01-15T14:30:00Z",
        "modified": "2025-01-15T14:30:00Z",
        "consistent": true
      }
    },
    "noiseAnalysis": {
      "pattern": "Canon EOS R5",
      "confidence": 94
    },
    "compressionAnalysis": {
      "doubleJPEG": false,
      "qualityEstimate": 95
    }
  },
  "reportUrl": "https://api.nftmv.io/reports/abc123.pdf",
  "differenceMapUrl": "https://api.nftmv.io/maps/abc123.png"
}
```

#### 3. Búsqueda y Exploración

**GET /api/v1/search**

Busca medios certificados.

```
Request:
GET /api/v1/search?q=protest&creator=0x742d...&from=2025-01-01&to=2025-01-31&type=image&limit=20&offset=0

Query Parameters:
- q: Búsqueda en título/descripción/tags
- creator: Wallet address del creador
- from: Fecha desde (ISO 8601)
- to: Fecha hasta
- type: image | video
- tags: Comma-separated tags
- location: Lat,lon,radius (e.g., "40.7128,-74.0060,10km")
- limit: Items per page (default 20, max 100)
- offset: Pagination offset

Response: 200 OK
{
  "results": [
    {
      "certificateId": "cert_...",
      "hash": "...",
      "thumbnail": "https://...",
      "title": "...",
      "creator": "0x...",
      "createdAt": "...",
      "tags": [...]
    }
  ],
  "total": 156,
  "limit": 20,
  "offset": 0,
  "hasMore": true
}
```

**GET /api/v1/media/:certificateId**

Obtiene detalles completos de un medio certificado.

```
Response: 200 OK
{
  "certificateId": "cert_a3f2b8d9",
  "hash": "a3f2b8d9...",
  "mediaType": "image/jpeg",
  "dimensions": {
    "width": 4000,
    "height": 3000
  },
  "fileSize": 2456789,
  "creator": {
    "address": "0x742d...",
    "name": "John Doe",
    "reputation": 95,
    "verified": true
  },
  "certification": {
    "certifiedAt": "2025-01-15T14:32:01Z",
    "blockchainTx": "0x8d4e7f2a...",
    "blockNumber": 12345678,
    "network": "polygon"
  },
  "storage": {
    "ipfsCID": "QmX8f3g2h...",
    "ipfsUrl": "https://ipfs.io/ipfs/QmX8f3g2h...",
    "cdnUrl": "https://cdn.nftmv.io/..."
  },
  "metadata": {
    "title": "...",
    "description": "...",
    "location": {...},
    "camera": "...",
    "capturedAt": "...",
    "tags": [...]
  },
  "authenticity": {
    "score": 95,
    "aiDetectionPassed": true,
    "manipulationDetectionPassed": true
  },
  "verification": {
    "totalVerifications": 1523,
    "lastVerified": "2025-01-20T10:15:00Z"
  },
  "revoked": false
}
```

**GET /api/v1/history/:hash**

Obtiene el historial completo de un medio (versiones, ediciones, etc.)

```
Response: 200 OK
{
  "hash": "a3f2b8d9...",
  "original": {
    "certificateId": "cert_a3f2b8d9",
    "createdAt": "2025-01-15T14:32:01Z",
    ...
  },
  "versions": [
    {
      "certificateId": "cert_b4g3c9e0",
      "createdAt": "2025-01-16T10:00:00Z",
      "changeType": "edited",
      "changes": "Color correction applied",
      "previousVersion": "cert_a3f2b8d9"
    }
  ],
  "verifications": [
    {
      "verifiedAt": "2025-01-15T15:00:00Z",
      "verifier": "anonymous",
      "status": "verified"
    },
    {
      "verifiedAt": "2025-01-16T12:30:00Z",
      "verifier": "fact-checker-org",
      "status": "verified",
      "notes": "Used in article XYZ"
    }
  ],
  "reports": []
}
```

#### 4. Usuario y Gestión

**GET /api/v1/user/me**

Obtiene perfil del usuario autenticado.

```
Response: 200 OK
{
  "address": "0x742d...",
  "name": "John Doe",
  "email": "john@example.com",
  "reputation": 95,
  "verified": true,
  "tier": "pro",
  "joined": "2024-06-01T00:00:00Z",
  "stats": {
    "totalCertifications": 1245,
    "totalVerifications": 523,
    "storageUsed": "15.4 GB"
  },
  "credits": {
    "remaining": 450,
    "resetDate": "2025-02-01T00:00:00Z"
  }
}
```

**GET /api/v1/user/:address/media**

Lista medios certificados por un usuario.

```
Response: 200 OK
{
  "creator": "0x742d...",
  "total": 1245,
  "media": [...]
}
```

#### 5. Reportes y Moderación

**POST /api/v1/report**

Reporta un medio sospechoso.

```
Request:
{
  "certificateId": "cert_a3f2b8d9",
  "reason": "suspected_alteration", // suspected_alteration | stolen_content | fake_metadata | unauthorized_use
  "description": "This image appears to have been edited after certification",
  "evidence": "https://..." // Optional URL to evidence
}

Response: 201 Created
{
  "reportId": "report_x9y8z7",
  "status": "submitted",
  "investigationUrl": "/api/v1/report/report_x9y8z7"
}
```

**GET /api/v1/report/:reportId**

Obtiene estado de un reporte.

```
Response: 200 OK
{
  "reportId": "report_x9y8z7",
  "status": "investigating", // submitted | investigating | resolved | dismissed
  "createdAt": "2025-01-20T10:00:00Z",
  "investigation": {
    "automaticAnalysisCompleted": true,
    "forensicScore": 72,
    "communityVotes": {
      "valid": 45,
      "invalid": 12
    }
  },
  "resolution": null
}
```

#### 6. Estadísticas y Analytics

**GET /api/v1/stats**

Estadísticas públicas de la plataforma.

```
Response: 200 OK
{
  "global": {
    "totalCertifications": 5234567,
    "totalVerifications": 52345678,
    "totalCreators": 45123,
    "totalStorageIPFS": "2.3 PB"
  },
  "recent": {
    "certificationsToday": 12456,
    "verificationsToday": 234567,
    "topCreators": [...]
  },
  "blockchain": {
    "network": "polygon",
    "totalTransactions": 5234567,
    "lastBlockNumber": 12345678
  }
}
```

### SDKs

#### JavaScript/TypeScript

```bash
npm install @nft-media-validator/sdk
```

```javascript
import { NFTMediaValidator } from '@nft-media-validator/sdk';

const client = new NFTMediaValidator({
  apiKey: 'your_api_key'
});

// Certificar
const result = await client.certify({
  media: fileBuffer,
  metadata: {
    title: 'Protest in Main Square',
    tags: ['protest', 'news']
  }
});

// Verificar
const verification = await client.verify({
  media: fileBuffer,
  level: 'forensic'
});

console.log(verification.status); // 'verified'
```

#### Python

```bash
pip install nft-media-validator
```

```python
from nft_media_validator import Client

client = Client(api_key='your_api_key')

# Certificar
result = client.certify(
    media_path='image.jpg',
    metadata={
        'title': 'Protest in Main Square',
        'tags': ['protest', 'news']
    }
)

# Verificar
verification = client.verify(
    media_path='image.jpg',
    level='forensic'
)

print(verification.status)  # 'verified'
```

#### Go

```bash
go get github.com/nft-media-validator/go-sdk
```

```go
import "github.com/nft-media-validator/go-sdk"

client := nftmv.NewClient("your_api_key")

// Certificar
result, err := client.Certify(&nftmv.CertifyRequest{
    Media: fileBytes,
    Metadata: nftmv.Metadata{
        Title: "Protest in Main Square",
        Tags:  []string{"protest", "news"},
    },
})

// Verificar
verification, err := client.Verify(&nftmv.VerifyRequest{
    Media: fileBytes,
    Level: nftmv.LevelForensic,
})
```

### Webhooks

**Configuración**:
```
Dashboard → Settings → Webhooks

Events:
- media.certified
- media.verified
- media.altered_detected
- media.reported
- report.resolved
```

**Payload Example**:
```json
{
  "event": "media.certified",
  "timestamp": "2025-01-15T14:32:01Z",
  "data": {
    "certificateId": "cert_a3f2b8d9",
    "hash": "a3f2b8d9...",
    "creator": "0x742d...",
    ...
  }
}
```

### Errores Estándar

```
Code    Error                     Message
───────────────────────────────────────────────────────
40001   INVALID_FORMAT            Unsupported file format
40002   FILE_TOO_LARGE            File exceeds size limit
40003   INVALID_METADATA          Metadata validation failed
40101   UNAUTHORIZED              Invalid or missing API key
40102   INSUFFICIENT_PERMISSIONS  Scope required: write:certify
40201   PAYMENT_REQUIRED          Insufficient credits
40401   NOT_FOUND                 Certificate not found
42201   AI_GENERATED_DETECTED     Content appears AI-generated
42202   MANIPULATED_CONTENT       Manipulation detected
42901   RATE_LIMIT_EXCEEDED       Too many requests
50001   INTERNAL_ERROR            Server error
50301   BLOCKCHAIN_UNAVAILABLE    Blockchain temporarily down
```

### OpenAPI Spec

Especificación completa disponible en:
```
https://api.nftmv.io/openapi.json
```

Documentación interactiva:
```
https://docs.nftmv.io/api
```

---

**API diseñada para desarrolladores. Documentada para humanos.**
