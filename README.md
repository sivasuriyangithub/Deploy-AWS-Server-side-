apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ template "jx.fullname" . }}-celery
  labels:
    app: {{ template "jx.name" . }}-celery
    chart: {{ template "jx.chart" . }}
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
  annotations:
    configmap.reloader.stakater.com/reload: "{{.Values.deployment.envConfigMapRef}}"
    secret.reloader.stakater.com/reload: "{{.Values.deployment.envSecretRef}}"
    {{- if .Values.deployment.postgresqlInNamespace }}
    secret.reloader.stakater.com/reload: "{{ .Release.Name }}-postgresql"
    {{- end }}
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      release: "{{ .Release.Name }}"
      app: {{ template "jx.name" . }}-celery
  template:
    metadata:
      labels:
        release: "{{ .Release.Name }}"
        chart: {{ template "jx.chart" . }}
        app: {{ template "jx.name" . }}-celery
    spec:
      terminationGracePeriodSeconds: {{ default 30 .Values.celery.gracePeriod }}
      {{- if .Values.allowPreemptible }}
      tolerations:
      - key: cloud.google.com/gke-preemptible
        operator: Equal
        value: "true"
        effect: NoSchedule
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - {{ template "jx.name" . }}-celery
              topologyKey: "kubernetes.io/hostname"
        {{- if .Values.preferPreemptible }}
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: cloud.google.com/gke-preemptible
                operator: Exists
            weight: 100
        {{- else if .Values.onlyPreemptible }}
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: cloud.google.com/gke-preemptible
                  operator: Exists
        {{- end }}
      {{- end }}
      volumes:
        - name: service-account
          secret:
            secretName: static-bucket-account
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          args:
            - "/app/start-celeryworker"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          volumeMounts:
            - name: service-account
              mountPath: "/etc/service_account_credentials"
              readOnly: true
          env:
          - name: IMAGE
            value: {{ .Values.image.repository }}
          - name: REVISION
            value: {{ .Values.image.tag }}
          - name: ENVIRONMENT_NAME
            value: {{ .Release.Namespace }}
          - name: PUBLIC_ORIGIN
            value: https://{{- template "ingress.url" . }}
          - name: DJANGO_ALLOWED_HOSTS
            value: "{{- template "ingress.url" . -}},{{ .Values.service.name }}"
          - name: DJANGO_SETTINGS_MODULE
            value: {{ .Values.deployment.settingsModule }}
          - name: GOOGLE_APPLICATION_CREDENTIALS
            value: "/etc/service_account_credentials/key.json"
          envFrom:
          {{- range .Values.deployment.envConfigMapRef }}
          - configMapRef:
              name: {{ . }}
          {{- end }}
          - secretRef:
              name: {{ .Values.deployment.envSecretRef }}
          resources:
            requests:
              cpu: {{ .Values.celery.cpuRequest }}
              memory: {{ .Values.celery.memoryRequest }}
            limits:
              cpu: {{ .Values.celery.cpuLimit }}
              memory: {{ .Values.celery.memoryLimit }}
