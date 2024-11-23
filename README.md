import telebot
import requests
import google.generativeai as genai
import os
import time
from telebot import types

# Configuración API KEY Gemini y Telegram
try:
    os.environ["API_KEY"] = "AIzaSyBmEsrcoQO36YNfPGizkdjIo1IVqaPzHUg"
    genai.configure(api_key=os.environ["API_KEY"])
except Exception as e:
    print(f"Error al configurar la API de Gemini: {e}")
    exit()

# Configuración de Gemini
try:
    model = genai.GenerativeModel('gemini-1.5-flash-latest')
except Exception as e:
    print(f"Error al crear el modelo de Gemini: {e}")
    exit()

# Token del bot de Telegram
TOKEN = "7972576195:AAFApKCt351YIzZzb3yjRkGCrqZVCGzAHrU"
bot = telebot.TeleBot(TOKEN)

# URL de la API de Remotive
API_URL = "https://remotive.com/api/remote-jobs"

import requests

# Diccionario de áreas de trabajo y localidades disponibles para los botones
AREAS_DE_TRABAJO = ['Design', 'Sales', 'Product', 'Customer Support', 'Marketing']

# Localidades que son válidas según la API de Remotive
LOCALIDADES = ['Worldwide', 'United States', 'UK', 'Canada', 'Germany', 'France']

# Variables para almacenar la selección de búsqueda
seleccion_localidad = None
seleccion_area_trabajo = None

# Variables para almacenar los trabajos mostrados
trabajos_mostrados_totales = []  # Para almacenar todos los trabajos mostrados

# Historial de conversación con Workie (máximo 3 interacciones)
historial_conversacion = {}

# Función que hace la búsqueda de empleos y muestra los primeros 5 resultados
def buscar_empleos(area_trabajo, localidad, mostrar_nuevos=True):
    global trabajos_mostrados_totales
    params = {
        'category': area_trabajo,  # Cambié 'title' por 'category' para usar el parámetro adecuado
        'location': localidad
    }

    try:
        response = requests.get(API_URL, params=params)
        response.raise_for_status()  # Lanza un error si la respuesta no es exitosa (status 200)
        data = response.json()
        jobs = data.get('jobs', [])

        # Filtramos para mostrar trabajos que no hayan sido mostrados previamente
        if mostrar_nuevos:
            nuevos_trabajos = [job for job in jobs if job not in trabajos_mostrados_totales]
        else:
            nuevos_trabajos = jobs

        if nuevos_trabajos:
            result = "Aquí tienes 5 búsquedas activas:\n"
            for i, job in enumerate(nuevos_trabajos[:5]):
                result += f"{i + 1}. ✨ Título: {job.get('title')}\n"
                result += f"   👉 Empresa: {job.get('company_name')}\n"
                result += f"   📍 Localidad: {job.get('candidate_required_location')}\n"
                result += f"   🔗 Enlace: {job['url']}\n"
                result += "-" * 5 + "\n"
            # Actualizamos la lista de trabajos mostrados
            trabajos_mostrados_totales.extend(nuevos_trabajos[:5])
            return result
        else:
            return "No se encontraron nuevos empleos para esta búsqueda.\n"
    except requests.exceptions.RequestException as e:
        print(f"Error al hacer la solicitud a la API de Remotive: {e}")
        return "Hubo un error al conectar con la API de empleos. Intenta de nuevo.\n"

# Función que muestra las opciones después de una respuesta de Workie
def mostrar_opciones_workie(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    markup.add(types.KeyboardButton("Seguir hablando con Workie 🤖"))
    markup.add(types.KeyboardButton("Buscar nuevos empleos 🔍"))
    markup.add(types.KeyboardButton("Cerrar Workapp 👋"))

    bot.send_message(
        message.chat.id,
        "¿Qué te gustaría hacer ahora?",
        reply_markup=markup
    )

    # Manejo de la opción "Buscar nuevos empleos 🔍"
    @bot.message_handler(func=lambda m: m.text == "Buscar nuevos empleos 🔍")
    def buscar_nuevos_empleos(message):
        # Redirige al flujo de selección de localidad (comienza de nuevo)
        # Reiniciamos las variables globales
        global seleccion_localidad, seleccion_area_trabajo
        seleccion_localidad = None
        seleccion_area_trabajo = None

        # Llamamos a la función de selección de localidad nuevamente
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
        localidad_buttons = [types.KeyboardButton(localidad) for localidad in LOCALIDADES]
        markup.add(*localidad_buttons)

        bot.send_message(
            message.chat.id,
            "¡Vamos a comenzar de nuevo! Primero, elige una localidad en la que quieras trabajar:",
            reply_markup=markup
        )

        # Registra el siguiente paso de la selección de localidad
        bot.register_next_step_handler(message, handle_localidad_selection)


# Función que muestra las opciones de búsqueda después de obtener resultados de trabajos
def mostrar_opciones(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    markup.add(types.KeyboardButton("Hablar con Workie 🤖"))
    markup.add(types.KeyboardButton("Buscar más empleos 🔍"))
    markup.add(types.KeyboardButton("Realizar una búsqueda nueva 💡"))
    markup.add(types.KeyboardButton("Cerrar Workapp 👋"))

    bot.send_message(
        message.chat.id,
        "¿Qué te gustaría hacer ahora? 🤔",
        reply_markup=markup
    )

# Función para interactuar con Gemini (con reintentos y manejo de tiempo)
def hablar_con_workie(pregunta, usuario_id):
    try:
        if usuario_id not in historial_conversacion:
            historial_conversacion[usuario_id] = []

        # Limitar el historial a 3 interacciones
        if len(historial_conversacion[usuario_id]) >= 3:
            historial_conversacion[usuario_id].pop(0)

        # Añadir la pregunta al historial
        historial_conversacion[usuario_id].append(f"Usuario: {pregunta}")

        # Crear el contexto para la conversación
        contexto = "\n".join(historial_conversacion[usuario_id])

        # Intentar obtener la respuesta de Gemini con un límite de tiempo
        timeout = 120  # Tiempo máximo de espera en segundos (2 minutos)
        start_time = time.time()
        while time.time() - start_time < timeout:
            try:
                respuesta = model.generate_content(contexto)
                # Añadir la respuesta de Workie al historial
                historial_conversacion[usuario_id].append(f"Workie: {respuesta.text}")
                return respuesta.text
            except Exception as e:
                print(f"Error al interactuar con Gemini: {e}")
                time.sleep(10)  # Intentar nuevamente después de 10 segundos
        # Si no se recibe respuesta en el tiempo límite
        return "Lo siento, parece que Workie está teniendo problemas para responder. Intenta de nuevo más tarde. 🕒"
    except Exception as e:
        print(f"Error al generar contenido con Gemini: {e}")
        return "Hubo un error al interactuar con Workie. Intenta de nuevo más tarde. 😞"

# Función que maneja el comando /start y muestra los botones de selección de localidad
@bot.message_handler(commands=['start'])
def enviar_bienvenida(message):
    # Crear un teclado con botones de localidades
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    localidad_buttons = [types.KeyboardButton(localidad) for localidad in LOCALIDADES]
    markup.add(*localidad_buttons)

    bot.send_message(
        message.chat.id,
        "¡Te damos la bienvenida a WorkApp! Vamos a ayudarte con tu búsqueda de trabajo. Primero, elige una localidad en la que quieras trabajar:",
        reply_markup=markup
    )

    # Guardamos el paso actual (localidad) para usarlo en la siguiente fase
    bot.register_next_step_handler(message, handle_localidad_selection)

# Función que maneja la selección de localidad
def handle_localidad_selection(message):
    global seleccion_localidad
    seleccion_localidad = message.text

    # Crear un teclado con botones de áreas de trabajo
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    area_buttons = [types.KeyboardButton(area) for area in AREAS_DE_TRABAJO]
    markup.add(*area_buttons)

    bot.send_message(
        message.chat.id,
        f"¡Genial! Has seleccionado la localidad: {seleccion_localidad}. Ahora, ¿en qué área quieres trabajar?:",
        reply_markup=markup
    )

    # Guardamos la localidad seleccionada para usarla después
    bot.register_next_step_handler(message, handle_area_selection)

# Función que maneja la selección de área de trabajo
def handle_area_selection(message):
    global seleccion_area_trabajo
    seleccion_area_trabajo = message.text

    # Realizar la búsqueda de empleos con la localidad y el área de trabajo seleccionados
    result = buscar_empleos(seleccion_area_trabajo, seleccion_localidad)
    bot.send_message(message.chat.id, result)

    # Mostrar opciones para continuar la conversación
    mostrar_opciones(message)

# Función que maneja las opciones seleccionadas por el usuario
@bot.message_handler(func=lambda message: message.text in ["Hablar con Workie 🤖", "Buscar más empleos 🔍", "Realizar una búsqueda nueva 💡", "Cerrar Workapp 👋"])
def manejar_opciones(message):
    global seleccion_localidad, seleccion_area_trabajo  # Asegurarnos de que las variables sean globales

    if message.text == "Hablar con Workie 🤖":
        # Iniciar conversación con Workie
        bot.send_message(message.chat.id, "¡Hola, soy Workie!🤖 La IA que te acompañará en tu búsqueda laboral, ¿en qué puedo ayudarte hoy?")
        bot.register_next_step_handler(message, procesar_pregunta_workie)

    elif message.text == "Buscar más empleos 🔍":
        # Muestra otros 5 trabajos con la misma búsqueda, sin repetir los ya mostrados anteriormente
        result = buscar_empleos(seleccion_area_trabajo, seleccion_localidad, mostrar_nuevos=True)
        bot.send_message(message.chat.id, result)

        # Muestra opciones nuevamente
        mostrar_opciones(message)

    elif message.text == "Realizar una búsqueda nueva 💡":
        # Al presionar "Realizar una búsqueda nueva", se vuelve a pedir la localidad y área
        handle_localidad_selection(message)

    elif message.text == "Cerrar Workapp 👋":
        # Finalizar la conversación
        bot.send_message(message.chat.id, "Qué lástima que tengamos que despedirnos.🥺 Gracias por usar WorkApp 💛 te esperamos para seguir acompañándote en tu búsqueda laboral. ¡Hasta pronto!👋")
        return

# Función para procesar preguntas a Workie (Gemini)
def procesar_pregunta_workie(message):
    pregunta = message.text
    respuesta = hablar_con_workie(pregunta, message.chat.id)
    bot.send_message(message.chat.id, respuesta)

    # Mostrar opciones para seguir conversando o finalizar
    mostrar_opciones_workie(message)

# Función que maneja la opción de seguir hablando con Workie
@bot.message_handler(func=lambda message: message.text == "Seguir hablando con Workie 🤖")
def seguir_hablando_con_workie(message):
    # Responder con la frase de continuación
    bot.send_message(message.chat.id, "¡Es una buena elección! ¿En qué más te puedo ayudar? 🤔")
    bot.register_next_step_handler(message, procesar_pregunta_workie)

# Inicia el bot
try:
    print("Iniciando bot de Telegram...")
    bot.polling()
except Exception as e:
    print(f"Error al iniciar el bot: {e}")

