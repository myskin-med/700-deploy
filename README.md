## 1. system update
```sh
sudo apt update
sudo apt upgrade
sudo /sbin/reboot
cd
cd go/src/github.com/myskin-med/duale-platform-v1/
```

## 2. duale fpaa integration setup:
```sh
nano .env

# add the following lines:
FPAA_DEV="app.duale.cloud:9999"
FPAA_PROD="app.duale.cloud:9999"
```
Note: in production, replace ip with app.duale.cloud


## 3-1. fpaa deploy
```sh
sudo systemctl stop fpaa.service
cd 
cd pyprojs
mv fpaa-pdf-fill/ fpaa-pdf-fill_bckp

git clone git@github.com:myskin-med/fpaa-pdf-fill.git
cd fpaa-pdf-fill
python3 -m venv .
source bin/activate
python -m pip install -r requirements.txt 
python main.py 
# CTRL-C to stop the server
deactivate
```

## 3-2. Test manual connection:
```
curl -k https://app.duale.cloud:9999/ping
```
Note: in production, avoid the -k flag



## 3-3. Set production mode
```sh
nano config/__init__.py
# set PROD_MODE = True
```

## 4. fpaa systemd setup:
```sh
sudo nano /etc/systemd/system/fpaa.service
```

```sh
[Unit]
# specifies metadata and dependencies
Description=Gunicorn instance to serve fpaa-pdf-fill
After=network.target
[Service]
User=myskin
Group=www-data
WorkingDirectory=/home/myskin/pyprojs/fpaa-pdf-fill
Environment="PATH=${PATH}:/home/myskin/pyprojs/fpaa-pdf-fill/bin"
Environment="DSN=dbname=msk_sw user=msk_user password=07R3BL4 port=5433"
Environment="FLASK_ENV=production"
ExecStart=/home/myskin/pyprojs/fpaa-pdf-fill/bin/gunicorn --timeout 0 --bind unix:/home/myskin/pyprojs/fpaa-pdf-fill/app.sock -m 007 main:app
[Install]
WantedBy=multi-user.target
```

## 5. start fpaa service
```sh
sudo systemctl daemon-reload
sudo systemctl start fpaa.service
sudo systemctl enable fpaa.service
sudo systemctl status fpaa.service

```
## 6. test fpaa connection:

```
curl -k https://app.duale.cloud:9999/ping
# respose: pong
```
Note: in production, replace ip with app.duale.cloud, then avoid the -k flag



## 7. migrations
```sh
psql -U msk_user -d msk_sw -p 5433
```

```sql
ALTER DATABASE msk_sw REFRESH COLLATION VERSION;

ALTER TABLE organizations
ADD COLUMN fpat text default '';
ALTER TABLE patients
ADD COLUMN fpaa text default '';
alter table staff_visit_services 
add column mcst text default '';
ALTER TABLE provisioned_services_onsite 
ADD column mcsa text default '';
ALTER TABLE patients
ADD COLUMN id_card text;
ALTER TABLE patients
RENAME COLUMN id_card TO id_card_1;
ALTER TABLE patients
ADD COLUMN id_card_2 text;
ALTER TABLE provisioned_services_onsite
ADD COLUMN mcsa_signed bool default 'f';

CREATE TABLE fea_journaling (
id SERIAL PRIMARY KEY,
fk_patient int REFERENCES patients(id) NOT NULL,
fk_organization int REFERENCES organizations(id) NOT NULL,
fpaa text NOT NULL,
id_card_1 text NOT NULL,
id_card_2 text,
full_name text NOT NULL,
fiscal_code text NOT NULL,
notes text,
ts_created timestamp default now(),
ts_updated timestamp default now()
);

UPDATE patients                                                
SET legal_tutor = legal_tutor || ARRAY['','']
WHERE legal_tutor IS NOT NULL;

ALTER TABLE patients ADD COLUMN mime_fpaa text;
ALTER TABLE patients ADD COLUMN mime_card_1 text;
ALTER TABLE patients ADD COLUMN mime_card_2 text;

ALTER TABLE fea_journaling ADD COLUMN mime_fpaa text;
ALTER TABLE fea_journaling ADD COLUMN mime_card_1 text;
ALTER TABLE fea_journaling ADD COLUMN  mime_card_2 text;

--^D to exit
```


## 8. deploy duale
```sh
cd
cd go/src/github.com/myskin-med/duale-platform-v1
git pull origin main
sudo -E make build
```

############################# 

############################################

####################################################################


### Extra ops on cloned server
# !!! DO NOT RUN IN PRODUCTION!!!
```
# mkdir /mnt/HC_Volume_10227180/assets/uploads/org_docs_renamed_avoid_copy_paste_accidentally
# mkdir /mnt/HC_Volume_10227180/assets/uploads/decr_buffer_renamed_avoid_copy_paste_accidentally
```
