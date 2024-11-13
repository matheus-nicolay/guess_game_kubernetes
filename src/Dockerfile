FROM python:3.8

WORKDIR /app

COPY . /app

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

ENV FLASK_APP="run.py"
ENV FLASK_ENV="production"  
ENV FLASK_DB_TYPE="postgres"
ENV FLASK_DB_USER="${FLASK_DB_USER:-postgres}"
ENV FLASK_DB_NAME="${FLASK_DB_NAME:-guess_game_db}"
ENV FLASK_DB_PASSWORD="${FLASK_DB_PASSWORD:-secretpass}"
ENV FLASK_DB_HOST="${FLASK_DB_HOST:-db}"
ENV FLASK_DB_PORT="${FLASK_DB_PORT:-5432}"

CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
