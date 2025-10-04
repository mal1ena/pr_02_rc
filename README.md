# pr_02_rc

## Лабораторная работа № 2. 
Проектирование и реализация клиент-серверной
системы. HTTP, веб-серверы и RESTful веб-сервисы
## Цель работы: 
изучить методы отправки и анализа HTTP-запросов с
использованием инструментов telnet и curl, освоить базовую настройку и
анализ работы HTTP-сервера nginx в качестве веб-сервера и обратного
прокси, а также изучить и применить на практике концепции архитектурного
стиля REST для создания веб-сервисов (API) на языке Python.

## Задание 1.
Проверка цепочки редиректов с http://mail.ru на https://mail.ru/ с помощью curl -v.

Делаем запрос curl -v http://mail.ru . Запрос падает в ошибку 301 и с помощью curl автоматически переходить на защищенный канал https, который уже в свою очередь передает html с информацией о странице.
<img width="1269" height="445" alt="image" src="https://github.com/user-attachments/assets/1184a79d-baeb-493c-aced-90d3bb2a4e2e" />

## Задание 2.
API для "Личные заметки" (сущность: id, title, content).

Устанавливаем Nginx
```
sudo apt install nginx -y
```
Запускаем и добавляем в автозагрузку
```
sudo systemctl start nginx
sudo systemctl enable nginx
```
Проверяем статус
```
sudo systemctl status nginx
```
<img width="867" height="508" alt="image" src="https://github.com/user-attachments/assets/542a5447-8545-49bc-863e-310326094af5" />

Видим статус active (running). Теперь, открываем в браузере http://localhost и видим приветственную страницу Nginx.
<img width="874" height="321" alt="image" src="https://github.com/user-attachments/assets/bc916cde-9fbe-4558-a046-fd036f747666" />

Устанавливаем систему управления виртуальными окружениями
```
sudo apt install python3-venv -y
```
Создаем директорию для нашего проекта
```
mkdir ~/rest_api_lab && cd ~/rest_api_lab
```
Создаем виртуальное окружение
```
python3 -m venv venv
```
Активация
```
source venv/bin/activate
```
В начале строки терминала видим venv

<img width="672" height="222" alt="image" src="https://github.com/user-attachments/assets/b5d8a850-7f99-4f28-85ad-9b521e3ec83c" />

Установливаем Flask.
```
pip install Flask
```
<img width="680" height="415" alt="image" src="https://github.com/user-attachments/assets/15273427-1ae1-4447-8f90-8ad6273bfaf1" />

Создаём файл app.py с помощью nano app.py и добавляем в него следующий код (маршруты для API придумываем сами). Этот API будет управлять списком задач.
```
from flask import Flask, jsonify, request

app = Flask(__name__)
app.config['JSON_AS_ASCII'] = False

notes = [
    {"id": 1, "title": "Первая заметка", "content": "Это содержание моей первой заметки."},
    {"id": 2, "title": "Идеи для проекта", "content": "Нужно изучить Docker и Kubernetes."}
]
next_id = 3

@app.route('/api/notes', methods=['GET'])
def get_notes():
    return jsonify({'notes': notes})

@app.route('/api/notes/<int:note_id>', methods=['GET'])
def get_note(note_id):
    note = next((note for note in notes if note['id'] == note_id), None)
    if note:
        return jsonify(note)
    return jsonify({'error': 'Note not found'}), 404

@app.route('/api/notes', methods=['POST'])
def create_note():
    global next_id
    if not request.json or not 'title' in request.json:
        return jsonify({'error': 'Title is required'}), 400

    new_note = {
        'id': next_id,
        'title': request.json['title'],
        'content': request.json.get('content', '')
    }
    notes.append(new_note)
    next_id += 1
    return jsonify(new_note), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```
<img width="678" height="711" alt="image" src="https://github.com/user-attachments/assets/26eb78ef-fc3d-4eb1-b9f8-c9e6376fdf14" />

Запускаем Flask-приложения и видим сообщение, что сервер запущен на http://0.0.0.0:5000/
<img width="486" height="273" alt="image" src="https://github.com/user-attachments/assets/15906dfc-e4a6-45d1-841d-efcb8c1c3297" />

Делаем запросы на получение всех заметок, одной заметки и на создание новой заметки
<img width="1240" height="286" alt="image" src="https://github.com/user-attachments/assets/b267667f-a7a3-4ea8-ad6a-8cf07135c724" />
<img width="799" height="137" alt="image" src="https://github.com/user-attachments/assets/c0d55504-93f6-4029-ab2b-d8b8df733a95" />
<img width="809" height="158" alt="image" src="https://github.com/user-attachments/assets/eeedda37-c83b-4026-965d-31ec056fcd14" />

Видим, что ответ получаем в виде Unicode escape sequences. Исправляем ситуацию с помощью установки jq и получаем ответ кириллицей

<img width="777" height="277" alt="image" src="https://github.com/user-attachments/assets/4bd93495-5a0d-49df-9de5-49774e1b613d" />

## Задание 3.
Настроить кеширование всех GET-запросов к API на 1 минуту.

Открываем конфигурационный файл Nginx.
```
sudo nano /etc/nginx/sites-available/default
```
и ищем блок location / { ... }, после него добавьте новый блок location /api/ { ... }.
```
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        
        proxy_cache api_cache;
        proxy_cache_valid 200 1m; # Кешировать успешные ответы на 1 минуту
        proxy_cache_methods GET;  # Кешировать только GET-запросы
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
<img width="415" height="789" alt="image" src="https://github.com/user-attachments/assets/b2c0cdee-cff0-40c2-b5ac-74873678b9f2" />

Перезапускаем Nginx
```
sudo nginx -t
```
Видим "syntax is ok" и перезапускаем:
```
sudo systemctl restart nginx
```
<img width="578" height="85" alt="image" src="https://github.com/user-attachments/assets/5a83d222-2de2-4520-8c27-6e818f2a2f2b" />

Теперь тестируем кеширование
Первый запрос попадает в кеш (MISS)
```
curl -I http://localhost/api/notes/2
```
Второй запрос берется из кеша (HIT)
```
curl -I http://localhost/api/notes/2
```
<img width="724" height="292" alt="image" src="https://github.com/user-attachments/assets/8040275b-73de-4833-adef-b05549e1fb6a" />
