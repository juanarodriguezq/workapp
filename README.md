#Bienvenidos al código de Workapp, el bot para buscar trabajo y hablar con Workie la IA al respecto :) 

import telebot
import requests
import google.generativeai as genai
import os
import time
from telebot import types

# Configuración API KEY Gemini y Telegram
try:
    os.environ["API_KEY"] = "AIzaSyCUmTcXHVi1PKGBcsP2Pye7OfWD1CEjcCI"
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
AREAS_DE_TRABAJO = [
    'Diseño', 'Ventas', 'Producto', 'Atención al cliente', 'Marketing'
]

# Localidades que son válidas según la API de Remotive
LOCALIDADES = [
    'En todo el mundo', 'Estados Unidos', 'Inglaterra', 'Canada', 'Alemania', 'Francia'
]

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
        'category':
        area_trabajo,  
        'location': localidad
    }

    try:
        print(params)
        response = requests.get(API_URL, params=params)
        print(response)
        response.raise_for_status(
        )  # Lanza un error si la respuesta no es exitosa (status 200)
        data = response.json()
        jobs = data.get('jobs', [])

        # Filtramos para mostrar trabajos que no hayan sido mostrados previamente
        if mostrar_nuevos:
            nuevos_trabajos = [
                job for job in jobs if job not in trabajos_mostrados_totales
            ]
        else:
            nuevos_trabajos = jobs

        if nuevos_trabajos:
            result = "¡Perfecto! Aquí tienes 5 búsquedas activas:\n"
            for i, job in enumerate(nuevos_trabajos[:5]):
                job_type = ''
                if job['job_type'] == 'full_time':
                    job_type = 'Full Time'
                elif job['job_type'] == 'part_time':
                    job_type = 'Part Time'
                elif job['job_type'] == 'contract':
                    job_type = 'Contrato'
                elif job['job_type'] == 'internship':
                    job_type = 'Pasantia'
                salary = job['salary'] if job['salary'] else "Sin especificar"
                result += f"{i + 1}. ✨ Título: {job['title']}\n"
                result += f"   👉 Empresa: {job['company_name']}\n"
                result += f"   💸 Salario: {salary}\n"
                result += f"   📍 Tipo de trabajo: {job_type}\n"
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
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True,
                                       one_time_keyboard=True)
    markup.add(types.KeyboardButton("Seguir hablando con Workie 🤖"))
    markup.add(types.KeyboardButton("Realizar nueva búsqueda 💡"))
    markup.add(types.KeyboardButton("Cerrar Workapp 👋"))

    bot.send_message(message.chat.id,
                     "¿Qué te gustaría hacer ahora?",
                     reply_markup=markup)

    # Registrar el próximo paso para manejar la opción seleccionada
    bot.register_next_step_handler(message, manejar_opciones_workie)


def manejar_opciones_workie(message):
    global seleccion_localidad, seleccion_area_trabajo

    if message.text == "Seguir hablando con Workie 🤖":
        # Continuar la conversación con Workie
        bot.send_message(message.chat.id,
                         "¡Genial! ¿Qué más te gustaría saber?")
        bot.register_next_step_handler(message, procesar_pregunta_workie)

    elif message.text == "Realizar nueva búsqueda 💡":
        # Volver a empezar la búsqueda, mostrar las localidades
    

        # Crear el teclado de localidades para elegir nuevamente
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True,
                                           one_time_keyboard=True)
        localidad_buttons = [
            types.KeyboardButton(localidad) for localidad in LOCALIDADES
        ]
        markup.add(*localidad_buttons)

        bot.send_message(message.chat.id,"¡Qué bueno! Vamos a empezar desde cero. ¿Desde qué localidad te gustaría trabajar?",
                        
                         reply_markup=markup)

        # Registrar el siguiente paso para manejar la selección de localidad
        bot.register_next_step_handler(message, handle_localidad_selection)

    elif message.text == "Cerrar Workapp 👋":
        bot.send_message(message.chat.id,
                         "Gracias por usar WorkApp 💛. ¡Te esperamos para continuar tu búsqueda laboral!")
    else:
        bot.send_message(
            message.chat.id,
            "No entendí tu respuesta. Por favor, selecciona una opción válida."
        )
        mostrar_opciones_workie(message)


# Función que muestra las opciones de búsqueda después de obtener resultados de trabajos
def mostrar_opciones(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True,
                                       one_time_keyboard=True)
    markup.add(types.KeyboardButton("Hablar con Workie 🤖"))
    markup.add(types.KeyboardButton("Buscar más empleos 🔍"))
    markup.add(types.KeyboardButton("Realizar una búsqueda nueva 💡"))
    markup.add(types.KeyboardButton("Cerrar Workapp 👋"))

    bot.send_message(message.chat.id,
                     "¿Qué te gustaría hacer ahora? 🤔",
                     reply_markup=markup)


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
                historial_conversacion[usuario_id].append(
                    f"Workie: {respuesta.text}")
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
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True,
                                       one_time_keyboard=True)
    localidad_buttons = [
        types.KeyboardButton(localidad) for localidad in LOCALIDADES
    ]
    markup.add(*localidad_buttons)

    bot.send_message(
        message.chat.id,
        "¡Te damos la bienvenida a WorkApp!💛 El bot que te ayuda con tu búsqueda de trabajo.🤓 Para empezar, elige desde qué localidad quieres trabajar",
        reply_markup=markup)

    # Guardamos el paso actual (localidad) para usarlo en la siguiente fase
    bot.register_next_step_handler(message, handle_localidad_selection)


# Función que maneja la selección de localidad
def handle_localidad_selection(message):
    global seleccion_localidad
    seleccion_localidad = message.text

    # Crear un teclado con botones de áreas de trabajo
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True,
                                       one_time_keyboard=True)
    area_buttons = [types.KeyboardButton(area) for area in AREAS_DE_TRABAJO]
    markup.add(*area_buttons)

    bot.send_message(
        message.chat.id,
        f"¡Genial! Te mostraremos empleos que estén habilitados para trabajar desde {seleccion_localidad}. Ahora, ¿en qué área quieres trabajar?",
        reply_markup=markup)

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
@bot.message_handler(func=lambda message: message.text in [
    "Hablar con Workie 🤖", "Buscar más empleos 🔍",
    "Realizar una búsqueda nueva 💡", "Cerrar Workapp 👋"
])
def manejar_opciones(message):
    global seleccion_localidad, seleccion_area_trabajo  # Asegurarnos de que las variables sean globales

    if message.text == "Hablar con Workie 🤖":
        # Iniciar conversación con Workie
        bot.send_message(
            message.chat.id,
            "¡Hola, soy Workie!🤖 La IA que te acompañará en tu búsqueda laboral, ¿en qué puedo ayudarte hoy?"
        )
        bot.register_next_step_handler(message, procesar_pregunta_workie)

    elif message.text == "Buscar más empleos 🔍":
        # Muestra otros 5 trabajos con la misma búsqueda, sin repetir los ya mostrados anteriormente
        result = buscar_empleos(seleccion_area_trabajo,
                                seleccion_localidad,
                                mostrar_nuevos=True)
        bot.send_message(message.chat.id, result)

        # Muestra opciones nuevamente
        mostrar_opciones(message)

    elif message.text == "Realizar una búsqueda nueva 💡":
        # Volver a empezar la búsqueda, mostrar las localidades

        # Crear el teclado de localidades para elegir nuevamente
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True,
                                           one_time_keyboard=True)
        localidad_buttons = [
            types.KeyboardButton(localidad) for localidad in LOCALIDADES
        ]
        markup.add(*localidad_buttons)

        bot.send_message(message.chat.id,"¡Qué bueno! Vamos a empezar desde cero. ¿Desde qué localidad te gustaría trabajar?",
                         
                         reply_markup=markup)
       

        # Registrar el siguiente paso para manejar la selección de localidad
        bot.register_next_step_handler(message, handle_localidad_selection)

    elif message.text == "Cerrar Workapp 👋":
        # Finalizar la conversación
        bot.send_message(
            message.chat.id,
            "Qué lástima que tengamos que despedirnos.🥺 Gracias por usar WorkApp 💛 Te esperamos para seguir acompañándote en tu búsqueda laboral. ¡Hasta pronto!👋"
        )
        return


# Función para procesar preguntas a Workie (Gemini)
def procesar_pregunta_workie(message):
    pregunta = message.text
    respuesta = hablar_con_workie(pregunta, message.chat.id)
    bot.send_message(message.chat.id, respuesta)

    # Mostrar opciones para seguir conversando o finalizar
    mostrar_opciones_workie(message)


# Función que maneja la opción de seguir hablando con Workie
@bot.message_handler(
    func=lambda message: message.text == "Seguir hablando con Workie 🤖")
def seguir_hablando_con_workie(message):
    # Responder con la frase de continuación
    bot.send_message(message.chat.id,
                     "¡Es una buena elección! ¿En qué más te puedo ayudar? 🤔")
    bot.register_next_step_handler(message, procesar_pregunta_workie)


# Inicia el bot
try:
    print("Iniciando bot de Telegram...")
    bot.polling()
except Exception as e:
    print(f"Error al iniciar el bot: {e}")

