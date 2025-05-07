# Despliegue de Basdonax AI RAG en OpenShift

## Preparación

Primero, asegúrate de tener instalado el cliente de OpenShift (`oc`) y estar conectado a tu clúster:

```bash
# Verificar la instalación de oc
oc version

# Iniciar sesión en el clúster de OpenShift
oc login --token=<tu-token> --server=<url-del-servidor>

# Crear un nuevo proyecto para la aplicación
oc new-project basdonax-rag
```

## Paso 1: Crear volúmenes persistentes

Necesitamos crear PersistentVolumeClaims para los datos de ChromaDB y Ollama:

```bash
# Crear PVC para ChromaDB
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: chroma-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF

# Crear PVC para Ollama (modelos)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-models-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF
```

## Paso 2: Desplegar ChromaDB

```bash
# Crear deployment para ChromaDB
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chroma
  labels:
    app: chroma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chroma
  template:
    metadata:
      labels:
        app: chroma
    spec:
      containers:
      - name: chroma
        image: chromadb/chroma:0.5.1.dev111
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: chroma-data
          mountPath: /chroma/.chroma/index
      volumes:
      - name: chroma-data
        persistentVolumeClaim:
          claimName: chroma-pvc
EOF

# Crear servicio para ChromaDB
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: chroma
spec:
  selector:
    app: chroma
  ports:
  - port: 8000
    targetPort: 8000
EOF
```

## Paso 3: Desplegar Ollama

Para Ollama, necesitamos considerar si usamos GPU o no. Usaré la versión sin GPU para mayor compatibilidad, pero incluiré comentarios sobre cómo habilitarla con GPU.

```bash
# Crear deployment para Ollama
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  labels:
    app: ollama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - containerPort: 11434
        env:
        - name: OLLAMA_MODELS
          value: /ollama/models
        volumeMounts:
        - name: ollama-models
          mountPath: /ollama/models
      volumes:
      - name: ollama-models
        persistentVolumeClaim:
          claimName: ollama-models-pvc
EOF

# Crear servicio para Ollama
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  selector:
    app: ollama
  ports:
  - port: 11434
    targetPort: 11434
EOF
```

### Nota sobre GPU en OpenShift:

Si deseas usar GPU en OpenShift, necesitarás:

1. Asegurarte de que tu clúster tenga nodos con GPU y el Operator NVIDIA GPU instalado
2. Modificar el deployment de Ollama para solicitar recursos GPU:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

## Paso 4: Construir y desplegar la UI

Primero, necesitamos construir la imagen de la UI y subirla a un registro accesible desde OpenShift.

```bash
# Asumiendo que estás en el directorio principal del proyecto
# Construir la imagen
oc new-build --binary --name=basdonax-ui -l app=basdonax-ui
oc start-build basdonax-ui --from-dir=./app --follow

# O si prefieres usar tu propio registro:
# docker build -t <tu-registro>/basdonax-ui:latest ./app
# docker push <tu-registro>/basdonax-ui:latest
```

Ahora, despliega la UI:

```bash
# Crear deployment para la UI
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
  labels:
    app: ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
      containers:
      - name: ui
        image: image-registry.openshift-image-registry.svc:5000/$(oc project -q)/basdonax-ui:latest
        # Si usas un registro externo:
        # image: <tu-registro>/basdonax-ui:latest
        ports:
        - containerPort: 8080
        env:
        - name: MODEL
          value: "phi3"  # Usa "llama3" si tienes GPU configurada
        - name: EMBEDDINGS_MODEL_NAME
          value: "all-MiniLM-L6-v2"
        - name: TARGET_SOURCE_CHUNKS
          value: "5"
EOF

# Crear servicio para la UI
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ui
spec:
  selector:
    app: ui
  ports:
  - port: 8080
    targetPort: 8080
EOF

# Crear ruta para acceder a la UI desde fuera del clúster
oc create route edge --service=ui ui
```

## Paso 5: Configuración post-despliegue

Después de que todo esté desplegado, necesitamos descargar el modelo de LLM. Conéctate al pod de Ollama:

```bash
# Obtener el nombre del pod
OLLAMA_POD=$(oc get pods -l app=ollama -o jsonpath='{.items[0].metadata.name}')

# Descargar el modelo phi3 (o llama3 si usas GPU)
oc exec $OLLAMA_POD -- ollama pull phi3
```

## Verificación del despliegue

```bash
# Verificar que todos los pods están en estado Running
oc get pods

# Obtener la URL para acceder a la aplicación
oc get route ui
```

## Instrucciones de uso

1. Accede a la aplicación usando la URL proporcionada por el comando `oc get route ui`
2. Navega a la pestaña "Archivos" para subir documentos a la base de conocimiento
3. Regresa a la página principal para realizar consultas sobre tus documentos

## Solución de problemas

Si encuentras problemas, puedes verificar los logs de cada componente:

```bash
# Logs de ChromaDB
oc logs $(oc get pods -l app=chroma -o jsonpath='{.items[0].metadata.name}')

# Logs de Ollama
oc logs $(oc get pods -l app=ollama -o jsonpath='{.items[0].metadata.name}')

# Logs de la UI
oc logs $(oc get pods -l app=ui -o jsonpath='{.items[0].metadata.name}')
```

## Configuraciones adicionales (opcionales)

### Configuración de límites de recursos

Para establecer límites de CPU y memoria para tus servicios:

```bash
oc set resources deployment ui --limits=memory=2Gi,cpu=1 --requests=memory=1Gi,cpu=500m
oc set resources deployment chroma --limits=memory=4Gi,cpu=2 --requests=memory=2Gi,cpu=1
oc set resources deployment ollama --limits=memory=8Gi,cpu=4 --requests=memory=4Gi,cpu=2
```

### Escalar servicios

Si necesitas más instancias del frontend:

```bash
oc scale deployment ui --replicas=3
```

### Configurar variables de entorno adicionales

Si necesitas ajustar la configuración:

```bash
oc set env deployment/ui VARIABLE_NOMBRE=valor
```