#######################################################
#       LUSCA - High Performance Configuration        #
#######################################################

acl all src 0.0.0.0/0
acl manager proto cache_object
acl localhost src 127.0.0.1/32
acl to_localhost dst 127.0.0.0/8

#########################################
# jaringan local eth1 atau eth2 jika ada
#########################################

#acl localnet src 192.168.1.0/24
#acl localnet src 192.168.2.0/24

acl SSL_ports port 443
acl Safe_ports port 80		# http
acl Safe_ports port 21		# ftp
acl Safe_ports port 443		# https
acl Safe_ports port 70		# gopher
acl Safe_ports port 210		# wais
acl Safe_ports port 1025-65535	# unregistered ports
acl Safe_ports port 280		# http-mgmt
acl Safe_ports port 488		# gss-http
acl Safe_ports port 591		# filemaker
acl Safe_ports port 777		# multiling http

acl CONNECT method CONNECT
acl dynamic urlpath_regex cgi-bin \?

http_access allow manager localhost
http_access deny manager
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost
http_access allow localnet
http_reply_access allow all
http_gzip on
http_gzip_types text/plain,text/html,text/xml,text/css,application/xml,application/xhtml+xml,application/rss+xml,application/javascript,application/x-javascript
http_access deny all

####################
# transparent proxy
####################

http_port 3128 transparent

######
# ZPH
######

zph_mode tos
zph_local 0x30
zph_parent 0
zph_option 136

################################
# cache peer untuk thundercache
################################


###############
# memory cache 
###############

cache_mem 32 MB                     # untuk cache_mem = memory : 16
maximum_object_size_in_memory 8 MB  # ini default maximal squid 

memory_replacement_policy heap GDSF
cache_replacement_policy heap LFUDA

#############################
# lokasi partisi cache proxy
#############################

cache_dir aufs /cache1 7000 32 256 max-size=7000
cache_dir aufs /cache2 7000 32 256 max-size=7000
cache_dir aufs /cache3 7000 32 256 max-size=7000


minimum_object_size 0 KB
maximum_object_size 256 MB          # untuk maximum_object_size = 1/2 X memory

cache_swap_low 98%
cache_swap_high 99%

access_log /var/log/squid/access.log
cache_log /var/log/squid/cache.log
cache_log /dev/null
cache_store_log /dev/null
redirect_rewrites_host_header off

logfile_rotate 10
emulate_httpd_log off
mime_table /usr/share/squid/mime.conf
pid_filename /var/run/squid.pid
log_fqdn off
buffered_logs off

####################
# storeurl 
####################

acl store_rewrite_list urlpath_regex \/(get_video|videoplayback\?id|videoplayback.*id) 
acl store_rewrite_list urlpath_regex \.(jp(e?g|e|2)|gif|png|tiff?|bmp|ico|flv|wmv|3gp|mp(4|3)|exe|msi|zip|on2|mar|swf)\?
acl store_rewrite_list_domain url_regex ^http:\/\/([a-zA-Z-]+[0-9-]+)\.[A-Za-z]*\.[A-Za-z]*
acl store_rewrite_list_domain url_regex (([a-z]{1,2}[0-9]{1,3})|([0-9]{1,3}[a-z]{1,2}))\.[a-z]*[0-9]?\.[a-z]{3}
acl store_rewrite_list_path urlpath_regex \.(jp(e?g|e|2)|gif|png|tiff?|bmp|ico|flv|avc|zip|mp3|3gp|rar|on2|mar|exe)$
acl store_rewrite_list_domain_CDN url_regex (khm|mt)[0-9]?.google.com 
acl store_rewrite_list_domain_CDN url_regex photos-[a-z].ak.fbcdn.net 
acl store_rewrite_list_domain_CDN url_regex \.rapidshare\.com.*\/[0-9]*\/.*\/[^\/]* 
acl store_rewrite_list_domain_CDN url_regex ^http:\/\/(www\.ziddu\.com.*\.[^\/]{3,4})\/(.*) 
acl store_rewrite_list_domain_CDN url_regex ^http:\/\/[.a-z0-9]*\.photobucket\.com.*\.[a-z]{3}$ 
acl store_rewrite_list_domain_CDN url_regex (khm|mt)[0-9]?.google.co(m|\.id)  
acl store_rewrite_list_domain_CDN url_regex streamate.doublepimp.com.*\.js\? \.doubleclick\.net.* yieldmanager cpxinteractive  quantserve\.com

acl dontrewrite url_regex yimg.com  redbot\.org (get_video|videoplayback\?id|videoplayback.*id).*begin\=[1-9][0-9]* \.php\? threadless.*\.jpg\?r=
acl getmethod method GET

storeurl_access deny dontrewrite
storeurl_access deny !getmethod

storeurl_access allow store_rewrite_list_domain_CDN
storeurl_access allow store_rewrite_list
storeurl_access allow store_rewrite_list_domain store_rewrite_list_path
storeurl_access deny all
storeurl_rewrite_program /etc/squid/storeurl.pl

##############################
# nilai penulisan ulang cache
##############################

storeurl_rewrite_children 5      # default squid ga perlu dirubah 
storeurl_rewrite_concurrency 100 # klo nilai minimal 10 sampai 500, 

max_stale 10 years
acl QUERY urlpath_regex -i \.(ini|ui|lst|inf|pak|ver|patch)$
acl QUERY urlpath_regex -i (dat.asp|afs.dat|notice.swf|patchlist.txt|hackshield|captcha|reset.css|update.ver|notice.html|updates.txt|gamenotice)
cache deny QUERY

##################
# refresh pattern
##################

refresh_pattern -i  \.(sc-|dl-|ex-|mh-|mst|dll)$                0  20% 0
refresh_pattern -i (main.exe|notice.html)$                      0  20% 0
refresh_pattern -i (livescore.com|UpdaterModifier.exe|FreeStyle.exe|FSLauncher.exe) 0  20% 0

refresh_pattern (get_video|videoplayback|videodownload|\.flv).*(begin|start)\=[1-9][0-9]*       0 0% 0
refresh_pattern imeem.*\.flv  0 0% 0  override-lastmod override-expire
refresh_pattern ^ftp: 40320     20%     40320   override-expire reload-into-ims store-stale
refresh_pattern ^gopher:        1440    0%      1440
refresh_pattern -i (livescore.com|UpdaterModifier.exe|FreeStyle.exe|FSLauncher.exe) 0  20% 0

#speedtest
refresh_pattern .speedtest.* 0 60% 10 negative-ttl=0
refresh_pattern speedtest.*\.(jp(e?g|e|2)|tiff?|bmp|gif|png|swf|txt|js) 0 50% 180 store-stale negative-ttl=0

#ads
refresh_pattern ^.*safebrowsing.*google   131400 999999% 525600 override-expire ignore-reload ignore-no-cache ignore-no-store ignore-private ignore-auth ignore-must-revalidate negative-ttl=10080 store-stale
refresh_pattern ^.*(streamate.doublepimp.com.*\.js\?|utm\.gif|ads\?|rmxads\.com|ad\.z5x\.net|bh\.contextweb\.com|bstats\.adbrite\.com|a1\.interclick\.com|ad\.trafficmp\.com|ads\.cubics\.com|ad\.xtendmedia\.com|\.googlesyndication\.com|advertising\.com|yieldmanager|game-advertising\.com|pixel\.quantserve\.com|adperium\.com|doubleclick\.net|adserving\.cpxinteractive\.com|syndication\.com|media.fastclick.net).* 5259487 20% 5259487 ignore-no-cache ignore-no-store ignore-private override-expire ignore-reload ignore-auth ignore-must-revalidate store-stale negative-ttl=40320 max-stale=1440

#antivirus
refresh_pattern avast.com.*\.vpx   40320 50% 525600 store-stale reload-into-ims
refresh_pattern (avgate|avira).*\.(idx|gz)$  1440 90% 1440  ignore-reload ignore-no-cache ignore-no-store store-stale ignore-must-revalidate 
refresh_pattern kaspersky.*\.avc$  131400 999999% 525600 ignore-reload store-stale
refresh_pattern kaspersky  1440 50% 131400 ignore-no-cache store-stale
refresh_pattern .symantecliveupdate.com.*\.zip  1440 90% 131400 ignore-must-revalidate store-stale
refresh_pattern .update.nai.com/.*\.(gem|zip|mcs) 43800 999999% 43800   ignore-reload store-stale ignore-must-revalidate
refresh_pattern .symantec.com.*\(exe|zip) 43800 999999% 43800   ignore-reload store-stale  ignore-must-revalidate
refresh_pattern ^http://file.pb.gemscool.com.*\.zip 131400 999999% 131400 override-expire store-stale
refresh_pattern ^http:\/\/\.www[0-9][0-9]\.indowebster\.com\/(.*)(rar|mov|mkv|cab|flv|wmv|3gp|mp(4|3)|exe|msi|zip) 43200 99999% 129600 reload-into-ims  ignore-reload override-expire ignore-no-cache ignore-no-store  ignore-private  store-stale ignore-auth

#kaskus
refresh_pattern .kaskus.us.*\.(jpg|gif|png) 1440 60% 131400 override-expire store-stale

#fb
refresh_pattern -i .facebook.com.*.(jpg|gif|png|swf|wav|mp(e?g|a|e|1|2|3|4)|3gp|flv|swf|wmv|zip|rar) 12960 999999% 129600
refresh_pattern -i .fbcdn.net.*.(jpg|gif|png|swf|wav|mp(e?g|a|e|1|2|3|4)|3gp|flv|swf|wmv|zip|rar) 12960 999999% 129690
refresh_pattern -i .zynga.com.*.(jpg|gif|png|swf|wav|mp(e?g|a|e|1|2|3|4)|3gp|flv|swf|wmv) 12960 999999% 129609
refresh_pattern -i .crowdstar.com.*.(jpg|gif|png|swf|wav|mp(e?g|a|e|1|2|3|4)|3gp|flv|swf|wmv) 12960 999999% 129609
refresh_pattern ^http://static.ak.fbcdn.net*.(jpg|gif|png|mp(e?g|a|e|1|2|3|4)|3gp|flv|swf|wmv) 129600 999999% 129600
refresh_pattern ^http://videoxl.l[0-9].facebook.com/(.*)(3gp|flv|swf|wmv|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600
refresh_pattern ^http://*.channel.facebook.com/(.*)(js|css|swf|jpg|gif|png|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600
refresh_pattern ^http://video.ak.facebook.com*.(3gp|flv|swf|wmv|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600
refresh_pattern ^http://photos-[a-z].ak.fbcdn.net/(.*)(css|swf|jpg|gif|png|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600
refresh_pattern ^http://profile.ak.fbcdn.net*.(jpg|gif|png) 129600 999999% 129600
refresh_pattern ^http://platform.ak.fbcdn.net/.* 720 100% 4320
refresh_pattern ^http://creative.ak.fbcdn.net/.* 720 100% 4320
refresh_pattern ^http://apps.facebook.com/.* 720 100% 4320
refresh_pattern ^http://static.ak.fbcdn.net*.(js|css|jpg|gif|png) 129600 999999% 129600
refresh_pattern ^http://statics.poker.static.zynga.com/(.*)(swf|jpg|gif|png|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600
refresh_pattern ^http://statics.poker.static.zynga.com/.* 720 100% 4320
refresh_pattern ^http://*.zynga.com*.(swf|jpg|gif|png|wav|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600
refresh_pattern ^http://*.crowdstar.com*.(swf|jpg|gif|png|wav|mp(e?g|a|e|1|2|3|4)) 129600 999999% 129600

# files
refresh_pattern -i \.(flv|x-flv|mov|avi|qt|mpg|mpeg)$ 10080 50% 43200 override-expire override-lastmod reload-into-ims ignore-reload ignore-no-cache  store-stale ignore-must-revalidate
refresh_pattern -i \.(swf|wav|mp3|mp4|au|mid)$ 10080 50% 43200 override-expire override-lastmod reload-into-ims ignore-reload ignore-no-cache ignore-auth  store-stale ignore-must-revalidate
refresh_pattern -i \.(iso|deb|rpm|zip|tar|tgz|ram|rar|bin|ppt|doc)$ 10080 90% 43200 ignore-no-cache ignore-auth store-stale ignore-must-revalidate
refresh_pattern -i \.(zip|gz|arj|lha|lzh)$ 10080 100% 43200 override-expire ignore-no-cache ignore-auth  store-stale ignore-must-revalidate
refresh_pattern -i \.(rar|tgz|tar|exe|bin)$ 10080 100% 43200 override-expire ignore-no-cache ignore-auth store-stale ignore-must-revalidate
refresh_pattern -i \.(hqx|pdf|rtf|doc)$ 10080 100% 43200 override-expire ignore-no-cache ignore-auth store-stale ignore-must-revalidate
refresh_pattern -i \.(inc|cab|ad|txt|dll|dl)$ 10080 100% 43200 override-expire ignore-no-cache ignore-auth store-stale ignore-must-revalidate

#specific sites
refresh_pattern \.rapidshare.*\/[0-9]*\/.*\/[^\/]* 131400 90% 525600 ignore-reload store-stale
refresh_pattern ^http://v\.okezone\.com/get_video\/([a-zA-Z0-9]) 131400 999999% 43200 override-expire ignore-reload ignore-no-cache ignore-no-store ignore-private ignore-auth override-lastmod ignore-must-revalidate negative-ttl=10080 store-stale
refresh_pattern (get_video\?|videoplayback\?|videodownload\?|\.flv?) 525600 99999999% 525600 override-expire ignore-reload ignore-no-cache ignore-must-revalidate ignore-private store-stale negative-ttl=0
refresh_pattern \.(ico|video-stats)  525600 999999% 525600 override-expire ignore-reload ignore-no-cache ignore-no-store ignore-private ignore-auth override-lastmod ignore-must-revalidate negative-ttl=10080 store-stale
refresh_pattern \.etology\?   525600 999999% 525600 override-expire ignore-reload ignore-no-cache store-stale
refresh_pattern galleries\.video(\?|sz)   525600 999999% 525600 override-expire ignore-reload ignore-no-cache store-stale
refresh_pattern brazzers\?   525600 999999% 525600 override-expire ignore-reload ignore-no-cache store-stale
refresh_pattern \.adtology\? 525600 999999% 525600 override-expire ignore-reload ignore-no-cache store-stale

refresh_pattern ^http://((cbk|mt|khm|mlt)[0-9]?)\.google\.co(m|\.id) 131400 999999% 525600 override-expire ignore-reload store-stale ignore-private negative-ttl=10080
refresh_pattern ytimg\.com.*\.(jpg|png) 525600 999999% 525600 override-expire ignore-reload store-stale
refresh_pattern images\.friendster\.com.*\.(png|gif)  131400 999999% 525600 override-expire ignore-reload store-stale
refresh_pattern garena\.com  525600 999999% 525600 override-expire reload-into-ims store-stale
refresh_pattern photobucket.*\.(jp(e?g|e|2)|tiff?|bmp|gif|png)  525600 999999% 525600 override-expire ignore-reload store-stale
refresh_pattern vid\.akm\.dailymotion\.com.*\.on2\?  525600 999999% 525600 ignore-no-cache override-expire override-lastmod store-stale
refresh_pattern ^http:\/\/images|pics|thumbs[0-9]\.  131400 999999% 525600 ignore-no-cache ignore-no-store ignore-reload override-expire store-stale
refresh_pattern ^http:\/\/www.onemanga.com.*\/   525600 999999% 525600 reload-into-ims override-expire store-stale
refresh_pattern mediafire.com\/images.*\.(jp(e?g|e|2)|tiff?|bmp|gif|png)  131400 999999% 525600 reload-into-ims override-expire ignore-private store-stale
refresh_pattern  \.macromedia.com.*\.(z|exe|cab)  131400 999999%  525600 ignore-reload override-expire  store-stale

#general
refresh_pattern \.(jp(e?g|e|2)|tiff?|bmp|gif|png) 131400 999999% 525600 ignore-no-cache ignore-no-store reload-into-ims override-expire ignore-must-revalidate store-stale
refresh_pattern \.(z(ip|[0-9]{2})|r(ar|[0-9]{2})|jar|bz2|gz|tar|rpm|vpu)  131400 999999% 525600 override-expire ignore-no-cache reload-into-ims
refresh_pattern \.(mp3|wav|og(g|a)|flac|midi?|rm|aac|wma|mka|ape)   131400 999999% 525600 override-expire reload-into-ims ignore-reload
refresh_pattern \.(exe|msi|dmg|bin|xpi|iso|swf|mar|psf|cab|mar)  131400 999999% 525600 override-expire reload-into-ims ignore-no-store ignore-no-cache ignore-must-revalidate
refresh_pattern \.(mkv|mpeg|ra?m|avi|mp(g|e|4)|mov|divx|asf|wmv|m\dv|rv|vob|asx|ogm|flv|3gp|on2)  525600 9999999% 525600 ignore-must-revalidate ignore-private ignore-no-cache override-expire reload-into-ims
refresh_pattern -i (cgi-bin) 0 0% 0
refresh_pattern \.(php|jsp|cgi|asx)\? 0 0% 0
refresh_pattern . 0 50% 525600 store-stale

quick_abort_min 0
quick_abort_max 0
quick_abort_pct 75

negative_ttl 2 minutes

############
# shoutcast
############

acl shoutcast rep_header X-HTTP09-First-Line ^ICY.[0-9]
upgrade_http0.9 deny shoutcast

icp_hit_stale on
vary_ignore_expire on
header_access X-Forwarded-For deny all

server_http11 on

half_closed_clients off
shutdown_lifetime 10 seconds

###################
# email dan domain
###################

cache_mgr ghost.ch4ndr4ya@yahoo.co.id
visible_hostname cjmedia.net
unique_hostname cjmedia.net

cache_effective_user proxy
cache_effective_group proxy
cachemgr_passwd none all

###############################
# bandwidth manager delaypools
###############################

acl warnet src 192.168.1.0/24
acl wifi src 192.168.2.0/24
acl operator src 192.168.1.27
acl blok url_regex -i \.wmv \.mpg \.mpeg \.wma \.wav \.3gp \.3gpp \.avi \.dat \.acc \.ogg \.mp4 \.mp3 \.mov \.rar \.zip \.7z \.iso \.ace \.exe \.torrent \.mkv \.rm \.asf \.flv \.pdf \.doc \.docx \.odt \.xls \.ppt \.pps \.dll \.vob \.tmp \.bin \.m4a \.xpi \.bup \.thm \.dmg  
acl blokftp url_regex ftp -i \.wmv \.mpg \.mpeg \.wma \.wav \.3gp \.3gpp \.avi \.dat \.acc \.ogg \.mp4 \.mp3 \.mov \.rar \.zip \.7z \.iso \.ace \.exe \.torrent \.mkv \.rm \.asf \.flv \.pdf \.doc \.docx \.odt \.xls \.ppt \.pps \.dll \.vob \.tmp \.bin \.m4a \.xpi \.bup \.thm \.dmg  
acl webvideo dstdomain -i \.youtube.com \.facebook.com \.myspace.com \.aol.com \.metacafe.com \.dailymotion.com \.vimeo.com \.Video.bing.com \.blip.tv \.break.com \.redtube.com \.Xtube.com \.youporn.com \.tube8.com \.Wrzuta.p \.games.co.id \.girlsgogames.co.id \.barbie.com \.dressupdollgames.net \.playpink.com
acl striming url_regex -i get_video? video_id? videodownload? videoplayback?
cache allow webvideo
cache allow striming
acl mimeUpload req_mime_type ^application/octet-stream$

delay_pools 5

delay_class 1 1
delay_parameters 1 -1/-1
delay_access 1 allow operator warnet
delay_access 1 deny all

delay_class 2 3
delay_parameters 2 24000/100000 16000/100000 8000/100000
delay_access 2 allow blok warnet
delay_access 2 allow blok wifi
delay_access 2 deny all

delay_class 3 3
delay_parameters 3 24000/100000 16000/100000 8000/100000
delay_access 3 allow blokftp warnet
delay_access 3 allow blokftp wifi
delay_access 3 deny all

delay_class 4 3
delay_parameters 4 24000/100000 16000/100000 8000/100000
delay_access 4 allow striming webvideo warnet
delay_access 4 allow striming webvideo wifi
delay_access 4 deny all

delay_class 5 3
delay_parameters 5 48000/100000 24000/100000 8000/100000
delay_access 5 allow mimeUpload warnet
delay_access 5 allow mimeUpload wifi

##############
# snmp
##############

snmp_port 3401
acl snmppublic snmp_community public
snmp_access allow snmppublic all

############
# icp port
############

icp_port 0
log_icp_queries off
query_icmp on

icon_directory /usr/share/squid/icons
error_directory /usr/share/squid/errors/English

nonhierarchical_direct on
prefer_direct off

max_filedescriptors 8192

ipcache_size 8192
ipcache_low 98
ipcache_high 99
fqdncache_size 8192

################
# MISCELLANEOUS
################

memory_pools off
forwarded_for off

##################
# lusca proxy
##################

client_db on
reload_into_ims on
coredump_dir /var/spool/squid/
pipeline_prefetch on
high_page_fault_warning 2
n_aiops_threads 24
load_check_stopen on
load_check_stcreate on
download_fastest_client_speed on







