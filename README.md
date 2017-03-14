# django_channels_example

#### This is very simple application to show you how to setup django-channels project
This app let you signup ,login , logout and it will show all login user and their 
status office/online

### Seting up environment

This is tested with python3.6 please recommend using 3.6 in ubuntu,

	$ sudo add repository ppa:jonathonf/python-3.6
	$ sudo apt update
	$ sudo apt install python3.6
	$ sudo apt install python3.6-dev


create virtual-environment

	$ virtualenv -p /usr/bin/python3.6 env
	$ source env/bin/activate
	$ git clone https://github.com/thisismsreddy/django_channels_example.git
	$ cd django_channels_example
	$ pip install -r requirement.txt

django-chanels requare to run redis server my case I am running redis server in a docker 

	$ docker run --name some-redis -p 6379:6379 --restart=always -d redis
	$ python manage.py makemigrations
	$ python manage.py migrate
	$ python manage.py runserver
	
	

