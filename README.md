Odoo电话营销模块基于Asterisk APP
====================

可在[Odoo](http://odoo.com)中直接打电话进行电话营销的[Asterisk IP-PBX](http://asterisk.org) 模块。

## 简介
This application is used to manage telemarketing campaigns e.g. call customers and playback a pre-recorded voice message or connect answered calls to operators. Other use cases are also possible as new features can be easily implemented using custom Asterisk dial plan.

This application uses Asterisk RESTful Interface (ARI) and requires **Asterisk v12** and newer. It is implemented inside Odoo using Python threads and does not have separate software components. 

*That's why You have to run Odoo in threaded mode devoted to one database disabling **workers** option and settings **dbfilter** option in Odoo configuration (see example of confoguration file below).*

## Software requirements and installation
需满足以下环境：

* 服务器需要安装[ari-py](https://github.com/asterisk/ari-py) (安装命令pip install ari)
* 基于[Odoo 8.0](https://github.com/odoo/odoo/tree/8.0/)（Ps：7.0的叫OpenERP，请参考https://www.odoo.com/apps/modules?price=Free&search=asterisk）
* Asterisk v.12/13 请将 func_curl 开启。

Installation:

* Install Odoo 8.0.
* Download asterisk_dialer from here and put it in your Odoo's addons folder. Or you can you my [install.sh](https://github.com/litnimax/odoo-asterisk-dialer/blob/master/install.sh) script to install Odoo & Asterisk dialer into virtual env in current directory.
* Install latest Asterisk. Versions < 12 do not have ARI support. 
* Install Asterisk ARI libs (pip install ari).

## Configuration

### Odoo configuration
In Odoo you should set **data_dir** option.
It is used as a root folder for storing uploaded voice files.
Be sure this folder is accessible for UID which is asterisk running under.

#### Example configuration file
```
[options]
addons_path = /home/max/tmp-work/odoo/odoo/openerp/addons,/home/max/tmp-work/odoo/myaddons,/home/max/tmp-work/odoo/odoo/addons,
admin_passwd = admin
data_dir = /home/max/tmp-work/odoo/filestore
db_host = localhost
db_password = openerp
db_user = openerp
dbfilter = ^odoo_dialer$
debug_mode = False
log_level = info
logfile = False
workers = 0
```
You can generate default Odoo configuration by running <code>./odoo -s</code>. It will create a default ~/.openerp_serverrc file in home directory.

**Warning!** Make sure You set up correct *dbfilter* option and also disabled workers by settings it to zero.

### Asterisk settings

#### ARI settings
ARI is configured in ari.conf. Example of configuration:

```
allowed_origins = *
[username]
type = user
read_only = no
password = $6$GPX.W2HVNvy9Bo$EeHySUu89U8.Wg6BvJCWNv51bDhu82t8gNz1u5n83MH1qWK282G2zV4V4neFldBRNb.nVchmRq28EGFTYl4QH.
password_format = crypt
```
Password is generated with mkpasswd -m sha-512 or just use password_format = plain at your risk ;-) and put plain password here. 

Remember it will be transfered over the network in plain text and if Asterisk is in internet using plain is a security hole. 

Imagine a phone bill for $40,000 for calls to Inmarsat because Asterisk ARI access is sniffed.

#### Dialplan

Dialer operates using ARI originate method and Local channel. Due to Aterisk limitation to return call status of non-connected calls we have to use Local channel and its context to actually send call to provider and use <code>h</code> exten to update call status. Here is an example of such a dial plan: 


```
[peer-1]
include => dialer_hangup
exten => _X!,1,Dial(SIP/${EXTEN}@peer-1,60,g)

[peer-2]
include => dialer_hangup
exten => _X!,1,Dial(SIP/${EXTEN}@peer-2,60,g)

[e1]
include => dialer_hangup
exten => _X!,1,Dial(DAHDI/g0/${EXTEN},60,g)

[dialer_hangup]
exten => h,1,Set(res=${CURL(http://localhost:8069/dialer/channel_update/?channel_id=${UNIQUEID}&status=${DIALSTATUS}&answered_time=${ANSWEREDTIME})})
;
;exten => h,n,Verbose(CALL ID: ${UNIQUEID}, DIAL STATUS: ${DIALSTATUS}, UPDATE RESULT: ${res})

```

So you must add the above snippet to your extensions.conf. Replace *peers* with your provider's peer from sip.conf and *localhost:8069* with your Odoo instance URL.

When creating Call Routing in Odoo Dialer Application name Dial Context exactly like Asterisk context name. 


### Running Dialer
Dialer operates in 2 modes (dialer type setting):

* Asterisk dialplan
* Odoo Stasis app

#### Asterisk Dialplan
When dialer type is set to *Playback* the Dialer originates calls and puts connected calls in specified Asterisk context name.

For example if instead of message playback we need to put every connected call in queue, the following dialplan must be created in extensions.conf:

```
[queue]
exten => _X.,1,Queue(test)
```
In Dialer configuration field *Context name* must be set to *queue*.

#### Stasis App
In this mode Dialer plays uploaded sound file to called number.

## Number Lists 
The Dialer can dial either Contacts (Partners) or custom list of phone numbers. This lists can be imported from .csv files.



## Troubleshooting
Enable debug mode.  

Run Odoo with ''--log-level=debug'' and see errors.

### Playback file is not played
Odoo saves sound files in a folder set by data_dir option. Check that Asterisk can read from there.


### Asterisk func curl not found
```
[Nov  5 11:13:25] ERROR[18838][C-000005e2]: pbx.c:4291 ast_func_read: Function CURL not registered
```
Install libcurl-devel, re-run ./configure and make menuselect, get sure res_curl and func_curl are selected and recompile and install these modules.

### Channel update script Not Found error
This is how this error looks in Odoo's log: 
```
2014-12-10 13:49:04,158 12709 INFO None openerp.http: Generating nondb routing
2014-12-10 13:49:04,298 12709 INFO None werkzeug: 127.0.0.1 - - [10/Dec/2014 13:49:04] "GET /dialer/channel_update/?channel_id=fe405e90-7fa8-11e4-bab0-70f395e579e2-1418132452-2&status=CHANUNAVAIL&answered_time=0 HTTP/1.1" 404 -

```
"Nondb routing" means there is either no database or more then one database available for selection. You have to check dbfilter option to be set correctly.

And this is how this error looks like in Asterisk console:
```
    -- Executing [h@e1:1] Set("Local/102@e1-00000001;2", "res=<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
    -- <title>404 Not Found</title>
    -- <h1>Not Found</h1>
    -- <p>The requested URL was not found on the server.  If you entered the URL manually please check your spelling and try again.</p>") in new stack
    -- Executing [h@e1:2] Verbose("Local/102@e1-00000001;2", "ID: f0e054c8-8073-11e4-bcc6-70f395e579e2-1418219617-2 DIAL STATUS: CHANUNAVAIL UPDATE RESULT: <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
    -- <title>404 Not Found</title>
    -- <h1>Not Found</h1>
    -- <p>The requested URL was not found on the server.  If you entered the URL manually please check your spelling and try again.</p>") in new stack
ID: f0e054c8-8073-11e4-bcc6-70f395e579e2-1418219617-2 DIAL STATUS: CHANUNAVAIL UPDATE RESULT: <!DOCTYPE HTML PUBLIC -//W3C//DTD HTML 3.2 Final//EN>
<title>404 Not Found</title>
<h1>Not Found</h1>
<p>The requested URL was not found on the server.  If you entered the URL manually please check your spelling and try again.</p>

```

### Asterisk permission issue
If you see something like that:
```
[Nov 23 16:54:57] WARNING[18025]: file.c:758 ast_openstream_full: File /home/openerp/.local/share/Odoo/filestore/odoo_production/sounds/01 does not exist in any format
[Nov 23 16:54:57] WARNING[18025]: file.c:1077 ast_streamfile: Unable to open /home/openerp/.local/share/Odoo/filestore/odoo_production/sounds/01 (format (slin)): Permission denied
[Nov 23 16:54:57] WARNING[18025]: res_stasis_playback.c:248 playback_final_update: a781d96a-7320-11e4-9f00-040108bd4001-1416754481-1: Playback failed for sound:/home/openerp/.local/share/Odoo/filestore/odoo_production/sounds/01
```
Make sounds folder accessible for Asterisk.

### Outdated python requests lib
```
File "/usr/local/lib/python2.7/dist-packages/swaggerpy/http_client.py", line 121, in __init__ self.auth = requests.auth.HTTPBasicAuth(username, password) 
AttributeError: 'module' object has no attribute 'HTTPBasicAuth'
```
python-requests package may be outdated. You may need to uninstall it and install a fresh one from python package repo (pip install requests --update).


## Feature requests 
The following features could be implemented if requested:

* Allow dialed person to press a key to confirm (aknowledge) the message
* Allow dialed person to be deleted from dialing list so that he will not be dialed on next round (like unsubscribe from list).
* Record dialed person choice in menu.
* Other.

To create a feature request [create](https://github.com/litnimax/asterisk_dialer/issues/new) a Github issue with label
*enhancement*.
