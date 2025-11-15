# Arquitectura Técnica

## Diseño del Sistema NFT Media Validator

### Visión General de la Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAPA DE CLIENTE                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Web    │  │  Mobile  │  │  Plugin  │  │   CLI    │       │
│  │   App    │  │   App    │  │  Adobe   │  │   Tool   │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                              │
│              REST API + GraphQL + WebSocket                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                    CAPA DE SERVICIOS                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │ Servicio   │  │ Servicio   │  │ Servicio   │               │
│  │ Certificac.│  │ Verificac. │  │ Análisis   │               │
│  └────────────┘  └────────────┘  └────────────┘               │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │ Servicio   │  │ Servicio   │  │ Servicio   │               │
│  │ Blockchain │  │ Estenograf.│  │ Metadatos  │               │
│  └────────────┘  └────────────┘  └────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ↑
┌─────────────────────────────────────────────────────────────────┐
│                      CAPA DE DATOS                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │ PostgreSQL │  │   IPFS     │  │ Blockchain │               │
│  │  Metadata  │  │   Media    │  │   Public   │               │
│  └────────────┘  └────────────┘  └────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

### Componentes Principales

#### 1. Capa de Cliente

**Aplicación Web**
- Interfaz principal para certificación y verificación
- Dashboard para gestión de medios certificados
- Visor de historial blockchain
- Tecnologías: React/Next.js, TailwindCSS

**Aplicación Móvil**
- Captura y certificación directa desde smartphone
- Verificación en tiempo real mediante cámara
- Modo offline con sincronización posterior
- Tecnologías: React Native / Flutter

**Plugins de Software**
- Integración con Adobe Lightroom/Photoshop
- Plugin para Final Cut Pro / Premiere
- Exportación directa con certificación
- Tecnologías: CEP (Common Extensibility Platform), APIs nativas

**CLI Tool**
- Herramienta de línea de comandos para automatización
- Scripts para procesamiento por lotes
- Integración con pipelines CI/CD de medios
- Tecnologías: Node.js / Go

#### 2. API Gateway

**Endpoints REST**
```
POST   /api/v1/certify          - Certificar nuevo medio
GET    /api/v1/verify/:hash     - Verificar medio
GET    /api/v1/media/:id        - Obtener información de medio
GET    /api/v1/history/:hash    - Historial completo
POST   /api/v1/validate         - Validar sello invisible
GET    /api/v1/user/:id/media   - Medios de usuario
```

**GraphQL API**
- Consultas flexibles para aplicaciones complejas
- Subscripciones en tiempo real para nuevas certificaciones
- Optimización de queries para reducir overhead

**WebSocket**
- Actualizaciones en tiempo real de estado de certificación
- Notificaciones de verificaciones de contenido propio
- Dashboard live de actividad de la red

#### 3. Servicio de Certificación

**Flujo de Procesamiento**

```javascript
1. Recepción de Medio
   - Validación de formato
   - Límites de tamaño
   - Escaneo de malware

2. Extracción de Metadatos
   - EXIF (cámara, fecha, ubicación)
   - XMP (derechos, descripción)
   - IPTC (periodístico)
   - Detección de ediciones previas

3. Análisis de Autenticidad
   - Detección de manipulación (ELA, JPEG artifacts)
   - Detección de contenido generado por IA
   - Análisis forense de sensor de cámara
   - Verificación de cadena de custodia digital

4. Generación de Sellos
   a) Sello Visible
      - Marca de agua personalizable
      - Posición: esquina inferior derecha
      - Incluye: Logo + ID único + QR verificación

   b) Sello Invisible
      - Método: DCT-based steganography
      - Payload: Hash + Timestamp + Creator ID
      - Robustez: Resistente a compresión JPEG hasta 70%

5. Cálculo de Hash Criptográfico
   - SHA-256 del contenido original
   - Hash del medio con sellos
   - Merkle tree para videos (hash por frame)

6. Registro en Blockchain
   - Transaction con metadatos
   - Hash del medio
   - Creator wallet address
   - Timestamp
   - IPFS CID del medio completo

7. Emisión de Certificado
   - Certificado digital JSON
   - Medio firmado
   - QR code de verificación
   - URL pública de verificación
```

**Tecnologías**
- Procesamiento: Python (PIL, OpenCV, FFmpeg)
- Detección IA: Modelos ML entrenados (CNN classifiers)
- Esteganografía: Algoritmos LSB + DCT + DWT
- Orquestación: Node.js/TypeScript

#### 4. Servicio de Verificación

**Proceso de Verificación**

```javascript
1. Recepción de Medio a Verificar

2. Extracción de Sello Invisible
   - Decodificación esteganográfica
   - Recuperación de payload
   - Extracción de hash y metadata

3. Cálculo de Hash Actual
   - Hash del medio recibido
   - Comparación con hash extraído

4. Consulta Blockchain
   - Búsqueda por hash
   - Verificación de existencia
   - Recuperación de metadatos originales

5. Análisis de Integridad
   - Comparación pixel-level (para imágenes)
   - Detección de manipulaciones post-certificación
   - Análisis de diferencias
   - Score de similitud

6. Generación de Reporte
   - Estado: Verificado / No Encontrado / Alterado
   - Detalles de certificación original
   - Fecha de certificación
   - Autor original
   - Cadena de custodia
   - Si alterado: mapa de diferencias
```

**Niveles de Verificación**

- **Nivel 1 - Rápido**: Solo consulta blockchain por hash (~100ms)
- **Nivel 2 - Estándar**: + Verificación de sello invisible (~1s)
- **Nivel 3 - Forense**: + Análisis completo de integridad (~10s)
- **Nivel 4 - Deep**: + Comparación con original en IPFS (~30s)

#### 5. Servicio de Blockchain

**Blockchain Selection**

Opciones evaluadas:
- **Ethereum**: Alta seguridad, costos de gas variables
- **Polygon**: Ethereum-compatible, bajos costos
- **Solana**: Alta velocidad, bajos costos
- **Avalanche**: Subnets personalizadas

**Recomendación Inicial**: Polygon
- Balance de seguridad y costo
- Compatibilidad Ethereum (EVM)
- Comunidad robusta
- Gas fees predecibles y bajos

**Smart Contract Estructura**

```solidity
contract MediaValidator {
    struct MediaCertificate {
        bytes32 contentHash;      // SHA-256 del contenido
        string ipfsCID;           // Contenido en IPFS
        address creator;          // Wallet del creador
        uint256 timestamp;        // Timestamp de certificación
        string mediaType;         // image/video
        string metadata;          // JSON metadata
        bool revoked;             // Flag de revocación
    }

    mapping(bytes32 => MediaCertificate) public certificates;
    mapping(address => bytes32[]) public creatorMedia;

    event MediaCertified(bytes32 indexed contentHash, address creator);
    event MediaRevoked(bytes32 indexed contentHash, string reason);

    function certifyMedia(...) external returns (bytes32)
    function verifyMedia(bytes32 hash) external view returns (MediaCertificate)
    function revokeMedia(bytes32 hash, string reason) external
    function getCreatorMedia(address creator) external view returns (bytes32[])
}
```

**Optimizaciones**
- Batch certification para reducir gas
- Layer 2 rollups para mayor escalabilidad
- Compresión de metadata
- Eventos indexados para queries eficientes

#### 6. Servicio de Esteganografía

**Algoritmos Implementados**

**Para Imágenes**

1. **DCT-based (Discrete Cosine Transform)**
   - Incrusta datos en coeficientes DCT
   - Similar a compresión JPEG
   - Resistente a compresión y filtros
   - Capacidad: ~1KB por imagen megapixel

2. **DWT-based (Discrete Wavelet Transform)**
   - Incrusta en subbandas de wavelets
   - Mayor robustez a ataques geométricos
   - Capacidad: ~500 bytes por megapixel

3. **LSB modificado (Least Significant Bit)**
   - Backup method, menor robustez
   - Mayor capacidad de payload
   - Usado para metadata extenso

**Para Videos**

1. **Frame-based DCT**
   - DCT en keyframes seleccionados
   - Balance de capacidad y robustez

2. **Motion Vector Embedding**
   - Datos en vectores de movimiento
   - Resistente a recompresión

3. **Audio Channel Embedding**
   - Esteganografía en canal de audio
   - Capa adicional de verificación

**Payload Estándar**

```json
{
  "version": "1.0",
  "hash": "sha256_hash_here",
  "certId": "unique_certificate_id",
  "timestamp": 1678901234,
  "creator": "0x...",
  "chain": "polygon",
  "txHash": "0x..."
}
```

#### 7. Almacenamiento

**PostgreSQL - Metadata**
- Información de usuarios
- Índice de medios certificados
- Logs de verificaciones
- Analytics y métricas
- Cache de queries blockchain

**IPFS - Media Content**
- Almacenamiento descentralizado de medios completos
- Redundancia automática
- Content-addressable (CID)
- Integración con Filecoin para persistencia garantizada

**Blockchain - Proof of Existence**
- Solo hashes y metadata mínimo
- Inmutabilidad
- Verificación independiente
- Auditabilidad pública

### Flujo de Datos Completo

**Certificación**
```
Cliente → API Gateway → Validación →
  → Procesamiento de Medio →
  → Análisis de Autenticidad →
  → Aplicación de Sellos →
  → Upload a IPFS → CID →
  → Cálculo de Hash →
  → Escritura en Blockchain → TxHash →
  → Guardado en PostgreSQL →
  → Generación de Certificado →
  ← Respuesta al Cliente (Certificado + Medio Firmado)
```

**Verificación**
```
Cliente → API Gateway → Extracción de Sello →
  → Consulta PostgreSQL (cache) →
  → Si no en cache: Consulta Blockchain →
  → Comparación de Hashes →
  → Análisis de Integridad →
  → Generación de Reporte →
  ← Respuesta al Cliente (Reporte de Verificación)
```

### Escalabilidad

**Estimaciones de Carga**

- **Certificaciones**: 10,000/día inicial → 1M/día objetivo
- **Verificaciones**: 100,000/día inicial → 10M/día objetivo
- **Almacenamiento**: ~1TB/mes medios → 100TB/año

**Estrategias de Escalado**

1. **Horizontal Scaling**
   - Microservicios containerizados (Docker/Kubernetes)
   - Load balancers para API Gateway
   - Procesamiento distribuido de medios

2. **Caching**
   - Redis para resultados de verificación frecuentes
   - CDN para medios certificados populares
   - Cache de queries blockchain

3. **Procesamiento Asíncrono**
   - Queue system (RabbitMQ/Kafka) para certificaciones
   - Workers paralelos para procesamiento
   - Status tracking en tiempo real

4. **Database Optimization**
   - Sharding por fecha de certificación
   - Índices optimizados para queries frecuentes
   - Read replicas para verificaciones

### Seguridad de la Infraestructura

- **API**: Rate limiting, autenticación JWT, validación de input
- **Procesamiento**: Sandbox para procesamiento de medios, escaneo de malware
- **Datos**: Encriptación en tránsito (TLS) y reposo (AES-256)
- **Blockchain**: Firma de transacciones, gestión segura de claves privadas
- **Backups**: Snapshots diarios, disaster recovery plan

### Monitoreo y Observabilidad

- **Logs**: Centralizados (ELK Stack / Grafana Loki)
- **Métricas**: Prometheus + Grafana
- **Tracing**: Distributed tracing (Jaeger/OpenTelemetry)
- **Alertas**: PagerDuty para incidencias críticas
- **Analytics**: Dashboard público de estadísticas de red

---

**Arquitectura diseñada para escalar con la verdad**
