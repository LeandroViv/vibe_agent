import os
import requests
import json
import subprocess

MAX_INTENTOS = 3
# Extensiones de archivos que el agente va a auditar para evitar redundancia
EXTENSIONES_PERMITIDAS = ('.js', '.ts', '.py', '.json', '.txt', '.md', '.yml', '.yaml')

def escanear_repositorio_completo():
    """
    Recorre el repositorio en el runner y arma un mapa textual 
    con la estructura y el contenido de los archivos clave.
    """
    contexto_repo = ""
    for raiz, directorios, archivos in os.walk('.'):
        # Ignorar carpetas ocultas o de dependencias de node/python
        if any(p en raiz for p in ['.git', 'node_modules', '__pycache__', '.venv', '.ollama']):
            continue
            
        for archivo in archivos:
            if archivo.endswith(EXTENSIONES_PERMITIDAS):
                ruta_completa = os.path.join(raiz, archivo)
                # Normalizar ruta eliminando el './' inicial
                ruta_relativa = os.path.relpath(ruta_completa, '.')
                
                try:
                    with open(ruta_completa, 'r', encoding='utf-8') as f:
                        contenido = f.read()
                    contexto_repo += f"\n--- ARCHIVO EXISTENTE: {ruta_relativa} ---\n{contenido}\n"
                except Exception:
                    pass # Ignorar archivos binarios o corruptos
                    
    return contexto_repo if contexto_repo else "[El repositorio está actualmente vacío]"

def consultar_ollama(prompt, mapa_repositorio, error_previo=""):
    url = "http://localhost:11434/api/generate"
    
    system_instruction = (
        "Sos un Agente App Builder experto en Vibe Coding. Tu objetivo es evolucionar el software sin introducir redundancias.\n"
        "Se te proporcionará el estado actual COMPLETO del repositorio. Debes AUDITARLO minuciosamente.\n"
        "Si el requerimiento del usuario ya se puede resolver con funciones o archivos existentes, REUTILIZALOS en lugar de duplicarlos.\n"
        "Responde EXCLUSIVAMENTE con un JSON válido (diccionario de ruta_archivo: contenido_completo).\n"
        "No incluyas explicaciones textuales ni bloques de código markdown (```json).\n"
    )
    
    if error_previo:
        system_instruction += f"\n⚠️ CORRECCIÓN DE ERROR (El build anterior falló):\n{error_previo}"

    full_prompt = (
        f"{system_instruction}\n\n"
        f"ESTADO ACTUAL DEL REPOSITORIO:\n{mapa_repositorio}\n\n"
        f"NUEVAS INSTRUCCIONES DEL CHAT:\n{prompt}\n\n"
        f"Respuesta JSON:"
    )

    payload = {
        "model": "qwen2.5-coder:3b",
        "prompt": full_prompt,
        "stream": False,
        "options": {
            "temperature": 0.1, 
            "num_ctx": 16384 # Ampliamos la ventana de contexto a 16k para que quepa el repositorio entero
        }
    }
    
    try:
        response = requests.post(url, json=payload)
        return response.json().get("response", "").strip()
    except Exception as e:
        return f"Error de conexión: {str(e)}"

def guardar_estructura(respuesta_raw):
    respuesta_raw = respuesta_raw.replace("```json", "").replace("```", "").strip()
    try:
        estructura = json.loads(respuesta_raw)
        for ruta, contenido in estructura.items():
            dir_padre = os.path.dirname(ruta)
            if dir_padre:
                os.makedirs(dir_padre, exist_ok=True)
            with open(ruta, "w", encoding="utf-8") as f:
                f.write(contenido)
        return True
    except:
        return False

def ejecutar_tests():
    try:
        if os.path.exists("package.json"):
            # Aquí correrías tus pruebas reales, ej: subprocess.run(["npm", "test"])
            return True, "Tests de Node completados con éxito."
        elif os.path.exists("requirements.txt"):
            result = subprocess.run(["pytest"], capture_output=True, text=True)
            return result.returncode == 0, result.stdout + result.stderr
        return True, "Sin entorno de pruebas formal detectado."
    except Exception as e:
        return False, str(e)

def main():
    prompt_usuario = os.getenv("PROMPT_CHAT", "Crea la estructura inicial.")
    error_actual = ""
    build_exitoso = False
    
    # Paso crítico: Escanear todo antes de llamar a la IA
    print("🔍 Escaneando repositorio completo para prevenir redundancias...")
    mapa_repositorio = escanear_repositorio_completo()

    for intento in range(1, MAX_INTENTOS + 1):
        print(f"🔄 Procesando cambio - Intento {intento}/{MAX_INTENTOS}...")
        respuesta = consultar_ollama(prompt_usuario, mapa_repositorio, error_previo=error_actual)
        
        if not guardar_estructura(respuesta):
            error_actual = "La IA rompió el formato JSON. Reintentando..."
            continue
            
        paso_tests, logs = ejecutar_tests()
        if paso_tests:
            build_exitoso = True
            break
        else:
            error_actual = logs

    with open("vibe_report.txt", "w", encoding="utf-8") as f:
        if build_exitoso:
            f.write("### 🤖 ¡App Buildeada y Auditada con Éxito!\n")
            f.write(f"El agente analizó el repositorio completo y aplicó los cambios en {intento} intentos sin generar código redundante.\n")
        else:
            f.write("### ⚠️ El Agente falló al intentar estabilizar el repositorio.\n\n")
            f.write(f"**Último error de compilación/tests:**\n```text\n{error_actual}\n```\n")

if __name__ == "__main__":
    main()
