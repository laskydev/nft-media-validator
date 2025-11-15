# Sistema de Validación

## Metodología Técnica para Verificación de Autenticidad

### Filosofía de Validación

> "No es suficiente que algo sea verdadero. Debe ser verificablemente verdadero."

La validación en NFT Media Validator opera en tres principios:

1. **Verificabilidad Independiente**: Cualquiera puede validar sin depender de nosotros
2. **Múltiples Fuentes de Verdad**: Combinación de evidencia técnica, criptográfica y forense
3. **Transparencia Total**: El proceso de validación es auditable y reproducible

### Niveles de Validación

#### Nivel 1: Verificación Básica (Blockchain Lookup)

**Tiempo**: ~100ms
**Costo**: Gratuito
**Casos de uso**: Verificación rápida en redes sociales, navegación casual

```
Proceso:
1. Extraer sello invisible del medio (si presente)
2. Calcular hash SHA-256 del contenido
3. Consultar blockchain por hash
4. Retornar: Certificado existe / No existe

Resultado:
✓ Certificado: Muestra metadatos básicos (autor, fecha, etc.)
✗ No Certificado: Medio no está en base de datos
```

**Limitaciones**:
- No detecta manipulaciones post-certificación
- No valida integridad del contenido
- Solo confirma que hash existe en blockchain

**API Endpoint**:
```
GET /api/v1/verify/quick/:hash
Response: { exists: true/false, certificate: {...} }
```

#### Nivel 2: Verificación Estándar (Integrity Check)

**Tiempo**: ~1-2s
**Costo**: Gratuito
**Casos de uso**: Verificación por periodistas, fact-checkers, público general

```
Proceso:
1. Extraer sello invisible completo
2. Validar integridad del sello (checksum)
3. Calcular hash del contenido actual
4. Consultar blockchain por hash del sello
5. Comparar hash actual vs hash certificado
6. Análisis básico de diferencias (si difieren)

Resultado:
✓ Verificado: Hash coincide 100%
⚠ Modificado Menor: Cambios detectados pero sello intacto (ej: recompresión)
✗ Alterado: Diferencias significativas
? No Encontrado: No certificado o sello removido
```

**Detección de Alteraciones**:
```
Si hash no coincide:
1. Calcular similitud perceptual (pHash)
2. Si similitud > 95%: "Modificado Menor"
   - Posibles causas: Recompresión, resize, ajustes color
3. Si similitud 80-95%: "Posiblemente Alterado"
   - Requiere inspección manual
4. Si similitud < 80%: "Alterado"
   - Cambios significativos detectados
```

**API Endpoint**:
```
POST /api/v1/verify/standard
Body: { media: <file/url> }
Response: {
  status: "verified" | "modified_minor" | "altered" | "not_found",
  certificate: {...},
  similarity: 0-100,
  differences: {...}
}
```

#### Nivel 3: Verificación Forense (Deep Analysis)

**Tiempo**: ~10-30s
**Costo**: Freemium (gratuito con límites, premium ilimitado)
**Casos de uso**: Investigaciones periodísticas, procesos legales, contenido crítico

```
Proceso:
1. Todo lo de Nivel 2
2. Análisis forense completo:
   a) Error Level Analysis (ELA)
   b) Clone detection
   c) Metadata forensics
   d) Noise pattern analysis
   e) Compression artifacts analysis
3. Descarga de original desde IPFS (si disponible)
4. Comparación pixel-by-pixel
5. Generación de mapa de diferencias
6. ML-based authenticity scoring

Resultado:
- Reporte forense detallado (PDF)
- Mapa visual de alteraciones
- Score de autenticidad (0-100)
- Evidencia técnica detallada
- Timeline de la cadena de custodia
```

**Mapa de Diferencias**:
```
Visualización:
- Heatmap de diferencias pixel-level
- Regiones sospechosas highlighted
- Métricas de distorsión por región
- Comparación side-by-side

Exportable en:
- PNG (overlay visual)
- JSON (datos técnicos)
- PDF (reporte completo)
```

**API Endpoint**:
```
POST /api/v1/verify/forensic
Body: { media: <file>, includeReport: true }
Response: {
  status: "...",
  certificate: {...},
  forensicAnalysis: {
    elaScore: 0-100,
    cloneDetection: {...},
    metadataConsistency: 0-100,
    authenticityScore: 0-100,
    alterationRegions: [...]
  },
  reportUrl: "https://..."
}
```

#### Nivel 4: Verificación Independiente (Self-Hosted)

**Tiempo**: Variable
**Costo**: Gratuito (requiere infraestructura propia)
**Casos de uso**: Organizaciones que requieren soberanía total, auditorías independientes

```
Herramientas Provistas:
1. CLI tool open source
2. Docker container con todas las dependencias
3. Scripts de consulta directa a blockchain
4. Algoritmos de extracción de sello invisible
5. Calculadoras de hash

Proceso:
1. Descargar herramientas
2. Ejecutar localmente sin depender de nuestra API
3. Consultar blockchain directamente
4. Verificar con algoritmos públicos

Ventajas:
- No depende de nuestros servidores
- Verificación incluso si la compañía desaparece
- Auditable completamente
- Sin límites de rate
```

**Ejemplo CLI**:
```bash
# Instalar
npm install -g @nft-media-validator/cli

# Verificar medio
nft-verify imagen.jpg

# Output:
# ✓ Certificado encontrado
# Hash: a3f2...
# Autor: 0x742d...
# Fecha: 2025-01-15 14:32:01 UTC
# Blockchain: Polygon
# TX: 0x8d4e...
# Integridad: 100% (sin alteraciones)
# Authenticity Score: 95/100

# Verificar con consulta directa a blockchain (sin API)
nft-verify imagen.jpg --direct-blockchain

# Verificación forense
nft-verify imagen.jpg --forensic --output report.pdf
```

### Criterios de Validación por Tipo

#### Imágenes (JPEG/PNG)

**Checklist de Validación**:

```
✓ Sello Invisible Presente
  - Payload extraíble y válido
  - Checksum correcto
  - Versión compatible

✓ Hash Coincidente
  - SHA-256 del contenido
  - Comparación con blockchain

✓ Metadata EXIF Consistente
  - Timestamps coherentes
  - Información de cámara válida (si presente)
  - GPS sin anomalías (si presente)
  - Software tags apropiados

✓ Análisis Forense Pasado
  - Sin evidencia de clonación
  - Sin double JPEG compression sospechosa
  - Noise pattern consistente
  - ELA sin anomalías graves

✓ Blockchain Verification
  - Hash registrado
  - Timestamp válido
  - Creador verificable
  - No revocado
```

**Puntuación de Autenticidad**:
```
100 pts: Totalmente verificado
 -5 pts: Recompresión post-certificación
-10 pts: Resize o recorte menor
-20 pts: Metadata alterado/removido
-30 pts: Evidencia de edición menor
-50 pts: Sello invisible dañado/parcial
-100 pts: Sello removido o hash no coincide
```

#### Videos (MP4/MOV/AVI)

**Checklist de Validación**:

```
✓ Sellos en Keyframes
  - Mínimo 5 keyframes con sello válido
  - Distribución uniforme a lo largo del video
  - Consistencia de payload entre frames

✓ Hash Merkle Tree
  - Hash de cada keyframe
  - Root hash coincide con blockchain
  - No frames faltantes

✓ Audio Channel Watermark (si aplicable)
  - Sello en canal de audio presente
  - Consistente con sello visual

✓ Metadata Consistente
  - Codec information
  - Frame rate, resolution
  - Timestamps de captura

✓ Análisis Forense
  - Sin evidencia de splicing (cortes/empalmes)
  - Motion vectors consistentes
  - Compression artifacts uniformes
  - Deepfake detection (para videos faciales)
```

**Validación por Segmento**:
```
Videos largos (>5 min):
- Validación de segmentos aleatorios (sampling)
- Al menos 10 puntos de verificación distribuidos
- Detección de ediciones en timeline
```

### Validación de Cadena de Custodia

**Tracking de Modificaciones Permitidas**

Algunas modificaciones NO invalidan el certificado:

```
Modificaciones Toleradas:
✓ Recompresión JPEG (quality > 70%)
✓ Resize proporcional (±20%)
✓ Conversión de formato (PNG ↔ JPEG)
✓ Ajustes menores de color/brillo (±10%)
✓ Adición de marca de agua adicional (no remove original)

Modificaciones que Invalidan:
✗ Cropping > 30%
✗ Remoción/alteración de marca visible
✗ Remoción de sello invisible
✗ Edición de contenido (clone, splice)
✗ Filtros pesados (deepfry, distorsión)
✗ Rotación > 5 grados
✗ Añadir/quitar elementos visuales
```

**Histórico de Versiones**:

```
Sistema de Re-Certificación:
Si un creador necesita editar contenido certificado:

1. Marca original como "Editado" en blockchain
2. Sube nueva versión con explicación de cambios
3. Nueva certificación con referencia a original
4. Blockchain mantiene enlace entre versiones

Resultado:
- Transparencia total de ediciones
- Original siempre accesible
- Historial de cambios inmutable
```

### Validación Comunitaria

#### Sistema de Reportes

**Cualquier usuario puede reportar**:
```
Tipos de Reporte:
1. "Sospecha de alteración"
   - Inicia análisis forense automático
   - Comparación con original

2. "Contenido robado"
   - Reclamo de autoría por tercero
   - Proceso de disputa

3. "Metadata falso"
   - Fecha/ubicación inconsistente con evento conocido

4. "Uso no autorizado"
   - Certificado legítimo pero uso sin permiso
```

**Proceso de Investigación**:
```
1. Reporte recibido
2. Análisis automatizado
   - Re-validación forense
   - Búsqueda de coincidencias en bases públicas
3. Si evidencia significativa:
   - Flagging temporal del contenido
   - Notificación al creador original
   - Período de respuesta (48h)
4. Resolución:
   - Comunitaria (votos ponderados por reputación)
   - Arbitraje (casos complejos)
   - Blockchain update (si procede revocación)
```

#### Sistema de Reputación

**Para Creadores**:
```
Score basado en:
+ Número de certificaciones válidas
+ Antigüedad de cuenta
+ Validaciones comunitarias positivas
- Reportes sustanciados
- Revocaciones

Beneficios de alta reputación:
- Badge de "Trusted Creator"
- Mayor visibilidad
- Acceso a features premium
```

**Para Verificadores**:
```
Score basado en:
+ Reportes acertados
+ Participación en disputas
- Reportes falsos/spam

Beneficios:
- Peso mayor en votos de resolución
- Acceso early a nuevas features
- Créditos para análisis forense
```

### Validación por IA y Crawlers

#### API para LLMs

```
Endpoint Optimizado:
GET /api/v1/ai/verify?url=<image_url>

Response (JSON-LD):
{
  "@context": "https://schema.org",
  "@type": "MediaObject",
  "contentUrl": "...",
  "verified": true,
  "verificationMethod": "NFT Media Validator",
  "datePublished": "2025-01-15",
  "creator": {
    "@type": "Person",
    "name": "John Doe",
    "identifier": "0x742d..."
  },
  "authenticity": {
    "score": 95,
    "certified": true,
    "alterations": "none"
  }
}

Uso por LLMs:
- GPT-4, Claude, etc. pueden consultar antes de usar imagen
- Citar autenticidad en respuestas
- Advertir sobre contenido no verificado
```

#### Crawlers de Fact-Checking

```
Robot.txt compatible:
- Permite crawling de endpoints de verificación
- Sitemap con certificados públicos
- Rate limits generosos para bots verificados

Integración con:
- Snopes
- FactCheck.org
- Politifact
- International Fact-Checking Network (IFCN)

Webhooks:
- Notificación en tiempo real de nuevas certificaciones
- Alertas de contenido relacionado a investigaciones activas
```

### Herramientas de Validación

#### Extensión de Navegador

```
Features:
- Verificación automática de imágenes en páginas
- Badge visual sobre imágenes certificadas
- Click derecho → "Verify with NFT Media Validator"
- Notificaciones de contenido no verificado en contextos críticos

Soportado:
- Chrome/Edge
- Firefox
- Safari
```

#### App Móvil de Verificación

```
Features:
- Escaneo mediante cámara (AR overlay)
- Upload desde galería
- Verificación de screenshots
- Historial de verificaciones
- Modo offline (cache de certificados comunes)
```

#### Plugin para Plataformas Sociales

```
Integración Propuesta:
Twitter/X:
- Badge en tweets con media certificado
- Warning en media viral no certificado

Facebook:
- Indicador en posts
- Integración con sistema de fact-checking existente

Instagram:
- Badge en posts de creadores verificados
- Stories con contenido certificado highlighted

TikTok:
- Indicador en videos certificados
- Búsqueda filtrable por contenido verificado
```

### Métricas de Validación

**Dashboard Público**:
```
Estadísticas en Tiempo Real:
- Total de verificaciones (hoy/mes/año)
- Tasa de verificación (certificado vs no certificado)
- Distribución de scores de autenticidad
- Top creadores por volumen
- Geografía de certificaciones
- Tipos de contenido más certificados

Transparencia:
- Todas las métricas públicas
- API de estadísticas abierta
- Datasets periódicos para investigación académica
```

---

**La validación es el corazón del sistema. Sin ella, solo tenemos metadata bonito.**
