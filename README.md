with open("Spy")
│
├── backend/
│   ├── app.py
│   ├── models.py
│   ├── routes.py
│   ├── notifications.py
│   ├── security.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/
│   ├── public/
│   │   ├── manifest.json
│   │   └── index.html
│   ├── src/
│   │   ├── App.js
│   │   ├── ThemeSwitcher.js
│   │   ├── RealtimeMap.js
│   │   ├── SharedLocation.js
│   │   ├── serviceWorker.js
│   │   └── i18n.js
│   ├── index.js
│   └── Dockerfile
│
└── docker-compose.yml

1. Backend (Flask)

1.1 app.py (Flask)

from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy
from flask_bcrypt import Bcrypt
from flask_jwt_extended import JWTManager
from flask_limiter import Limiter
from flask_socketio import SocketIO, emit
from redis import Redis
from security import verify_captcha, register_biometric, log_audit_event
from notifications import send_push_notification
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///spy.db'
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
app.config['JWT_SECRET_KEY'] = os.getenv('JWT_SECRET_KEY')

# Extensions Flask
db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)
socketio = SocketIO(app)
limiter = Limiter(app, key_func=lambda: request.remote_addr)
cache = Redis(host='localhost', port=6379)

# Importer les routes depuis un fichier séparé
from routes import auth_blueprint, location_blueprint
app.register_blueprint(auth_blueprint)
app.register_blueprint(location_blueprint)

if __name__ == "__main__":
    socketio.run(app, host="0.0.0.0", port=5000, debug=True)

1.2 models.py

from datetime import datetime
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(150), unique=True, nullable=False)
    phone_number = db.Column(db.String(20), unique=True, nullable=False)
    verified_email = db.Column(db.Boolean, default=False)
    verified_phone = db.Column(db.Boolean, default=False)
    location = db.relationship('Location', backref='user', lazy=True)

class Location(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    latitude = db.Column(db.Float, nullable=False)
    longitude = db.Column(db.Float, nullable=False)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

1.3 routes.py

from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required, create_access_token, get_jwt_identity
from models import User, Location, db
from security import log_audit_event
from notifications import send_push_notification

auth_blueprint = Blueprint('auth', __name__)
location_blueprint = Blueprint('location', __name__)

@auth_blueprint.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    email = data['email']
    phone_number = data['phone_number']
    user = User(email=email, phone_number=phone_number)
    db.session.add(user)
    db.session.commit()
    access_token = create_access_token(identity=user.id)
    return jsonify({'access_token': access_token})

@location_blueprint.route('/share_location', methods=['POST'])
@jwt_required()
def share_location():
    user_id = get_jwt_identity()
    data = request.get_json()
    latitude = data['latitude']
    longitude = data['longitude']
    location = Location(latitude=latitude, longitude=longitude, user_id=user_id)
    db.session.add(location)
    db.session.commit()

    # Envoie des notifications en temps réel via WebSockets
    emit('location_shared', {'user_id': user_id, 'latitude': latitude, 'longitude': longitude}, broadcast=True)

    return jsonify({'message': 'Location shared successfully.'}), 200

@location_blueprint.route('/get_location/<user_id>', methods=['GET'])
@jwt_required()
def get_location(user_id):
    locations = Location.query.filter_by(user_id=user_id).all()
    result = [{'latitude': loc.latitude, 'longitude': loc.longitude, 'timestamp': loc.timestamp} for loc in locations]
    return jsonify(result), 200

1.4 notifications.py (Gestion des notifications push)

from pywebpush import webpush, WebPushException
import os

def send_push_notification(subscription_info, data):
    try:
        webpush(
            subscription_info=subscription_info,
            data=data,
            vapid_private_key=os.getenv('VAPID_PRIVATE_KEY'),
            vapid_claims={"sub": "mailto:votre-email@example.com"}
        )
    except WebPushException as ex:
        print(f"Erreur lors de l'envoi de la notification: {ex}")

1.5 security.py

import requests
from datetime import datetime

def verify_captcha(captcha_response):
    secret = os.getenv('RECAPTCHA_SECRET_KEY')
    payload = {'secret': secret, 'response': captcha_response}
    response = requests.post('https://www.google.com/recaptcha/api/siteverify', data=payload)
    return response.json().get('success', False)

def register_biometric(user_id):
    # Logique pour l'authentification biométrique
    pass

def log_audit_event(user_id, action, description):
    with open('audit.log', 'a') as f:
        f.write(f'{datetime.utcnow()} | User {user_id} | Action: {action} | {description}\n')

1.6 Dockerfile (Backend)

FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY . /app

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]

1.7 requirements.txt

Flask
Flask-SQLAlchemy
Flask-JWT-Extended
Flask-Bcrypt
Flask-Limiter
Flask-SocketIO
redis
pywebpush
requests
gunicorn

2. Frontend (React)

2.1 App.js (React avec gestion des partages de localisation et WebSockets)

import React, { useState, useEffect } from 'react';
import { RealtimeMap } from './RealtimeMap';
import { ThemeSwitcher } from './ThemeSwitcher';
import { useTranslation } from 'react-i18next';
import './i18n';

function App() {
  const { t } = useTranslation();
  const [position, setPosition] = useState({ latitude: null, longitude: null });

  useEffect(() => {
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        setPosition({ latitude: pos.coords.latitude, longitude: pos.coords.longitude });
      },
      (err) => console.error(err),
      { enableHighAccuracy: true }
    );
  }, []);

  return (
    <div className="App">
      <ThemeSwitcher />
      <h1>{t('welcome_to_spy')}</h1>
      <RealtimeMap position={position} />
    </div>
  );
}

export default App;

2.2 RealtimeMap.js (Carte interactive en temps réel)

import React, { useEffect } from 'react';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

export function RealtimeMap({ position }) {
  useEffect(() => {
    const map = L.map('map').setView([position.latitude, position.longitude], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    L.marker([position.latitude, position.longitude]).addTo(map).bindPopup('Votre position').openPopup();
  }, [position]);

  return <div id="map" style={{ height: '400px', width: '100%' }}></div>;
}

2.3 ThemeSwitcher.js (Sélecteur de thème clair/sombre)

2.3 ThemeSwitcher.js (Sélecteur de thème clair/sombre)

import React, { useState } from 'react';

export function ThemeSwitcher() {
  const [theme, setTheme] = useState('light');

  const toggleTheme = () => {
    const newTheme = theme === 'light' ? 'dark' : 'light';
    setTheme(newTheme);
    document.documentElement.setAttribute('data-theme', newTheme);
  };

  return (
    <div>
      <button onClick={toggleTheme}>
        {theme === 'light' ? 'Passer en mode sombre' : 'Passer en mode clair'}
      </button>
    </div>
  );
}

2.4 i18n.js (Gestion des langues avec react-i18next)

import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import translationEN from './locales/en/translation.json';
import translationFR from './locales/fr/translation.json';
import translationES from './locales/es/translation.json';

const resources = {
  en: { translation: translationEN },
  fr: { translation: translationFR },
  es: { translation: translationES },
};

i18n.use(initReactI18next).init({
  resources,
  lng: 'en', // Langue par défaut
  fallbackLng: 'en',
  interpolation: {
    escapeValue: false,
  },
});

export default i18n;

2.5 serviceWorker.js (Service Worker pour les notifications et PWA)

const isLocalhost = Boolean(
  window.location.hostname === 'localhost' ||
    window.location.hostname === '[::1]' ||
    window.location.hostname.match(
      /^127(?:\.(?:25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])){3}$/
    )
);

export function register(config) {
  if (process.env.NODE_ENV === 'production' && 'serviceWorker' in navigator) {
    const publicUrl = new URL(process.env.PUBLIC_URL, window.location.href);
    if (publicUrl.origin !== window.location.origin) {
      return;
    }

    window.addEventListener('load', () => {
      const swUrl = `${process.env.PUBLIC_URL}/service-worker.js`;

      if (isLocalhost) {
        checkValidServiceWorker(swUrl, config);
      } else {
        registerValidSW(swUrl, config);
      }
    });
  }
}

function registerValidSW(swUrl, config) {
  navigator.serviceWorker
    .register(swUrl)
    .then((registration) => {
      registration.onupdatefound = () => {
        const installingWorker = registration.installing;
        if (installingWorker == null) {
          return;
        }
        installingWorker.onstatechange = () => {
          if (installingWorker.state === 'installed') {
            if (navigator.serviceWorker.controller) {
              if (config && config.onUpdate) {
                config.onUpdate(registration);
              }
            } else {
              if (config && config.onSuccess) {
                config.onSuccess(registration);
              }
            }
          }
        };
      };
    })
    .catch((error) => {
      console.error('Erreur lors de l\'enregistrement du service worker:', error);
    });
}

function checkValidServiceWorker(swUrl, config) {
  fetch(swUrl, {
    headers: { 'Service-Worker': 'script' },
  })
    .then((response) => {
      const contentType = response.headers.get('content-type');
      if (response.status === 404 || (contentType != null && contentType.indexOf('javascript') === -1)) {
        navigator.serviceWorker.ready.then((registration) => {
          registration.unregister().then(() => {
            window.location.reload();
          });
        });
      } else {
        registerValidSW(swUrl, config);
      }
    })
    .catch(() => {
      console.log('Pas de connexion internet. L\'application fonctionne en mode hors ligne.');
    });
}

export function unregister() {
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.ready.then((registration) => {
      registration.unregister();
    });
  }
}

2.6 manifest.json (PWA Configuration)

{
  "short_name": "Spy",
  "name": "Spy App",
  "icons": [
    {
      "src": "icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ],
  "start_url": "/",
  "background_color": "#ffffff",
  "theme_color": "#007bff",
  "display": "standalone",
  "orientation": "portrait"
}

2.7 index.html (Référence au manifest et au Service Worker)

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Spy</title>
    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>

2.8 Dockerfile (Frontend)

FROM node:14-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY . /app

CMD ["npm", "start"]

2.9 index.js (React Application Root avec Service Worker)

import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';
import * as serviceWorker from './serviceWorker';

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);

// Enregistrement du service worker pour activer le mode PWA
serviceWorker.register();

3. docker-compose.yml

Ce fichier docker-compose.yml permet de déployer à la fois le backend Flask et le frontend React ensemble, facilitant ainsi l’orchestration des conteneurs.

version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    env_file:
      - ./backend/.env
    networks:
      - spy-network
    depends_on:
      - redis

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    networks:
      - spy-network

  redis:
    image: "redis:alpine"
    networks:
      - spy-network

networks:
  spy-network:
    driver: bridge
