# start from an official image
FROM python:3

RUN mkdir -p /app/src/
WORKDIR /app/src/

# install our two dependencies
ADD requirements.txt /app/src/
RUN pip install -r requirements.txt

# copy our project code
COPY . /app/src/

# expose the port 8000
EXPOSE 8000

# define the default command to run when starting the container
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
#CMD ["gunicorn", "--bind", ":8000", "backend.wsgi:application"]
