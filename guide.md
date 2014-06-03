Tested requirements:
- Ubuntu 14.04 LTS amd64
- PostgreSQL 9.3.4
- lf-core 3.0.1
- webmcp 1.2.5
- rocketwiki-lqfb 0.4
- lf-frontend 2.2.5
- lighttpd 1.4.33-1+nmu2ubuntu2

Add postgresql repository (check distro version). Create `/etc/apt/sources.list.d/pgdg.list`. Add there `deb http://apt.postgresql.org/pub/repos/apt/ trusty-pgdg main`

    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

Install debian security updates:

    apt-get update
    apt-get upgrade
    
Install the neccessary debian packages:

    apt-get install lua5.1 postgresql postgresql-server-${version}-dev build-essential libpq-dev liblua5.1-0-dev lighttpd ghc libghc-parsec3-dev imagemagick exim4 nano

> +500MB

Create a database role for www-data:

	createuser www-data
    passwd postgres
    su - postgres
	psql
	postgres-# ALTER ROLE postgres WITH PASSWORD 'postgres';
	postgres-# CREATE USER liquid WITH PASSWORD 'feedback';
	postgres-# CREATE DATABASE lf;
	postgres=# GRANT ALL PRIVILEGES ON DATABASE lf to liquid;
	postgres=# \q
    exit

Make `www-data` user possible to login with. Edit `/etc/passwd`. Find *www-data* entry and change shell from `/usr/sbin/nologin` to `/bin/bash` (or what you prefer)

Configure db connection. Edit `/etc/postgres/${version}/main/pg_hba.conf`, change `peer` to `md5`

    local all postgres md5
    # TYPE DATABASE USER ADDRESS METHOD
    # "local" is for Unix domain socket connections only
    local all  all md5

Reload postgres configuration:


    service postgresql reload
Create a directory for unpacking source tarballs:

    cd /root
    mkdir install
Install and configure LiquidFeedback Core:

    cd /root/install
    wget http://www.public-software-group.org/pub/projects/liquid_feedback/backend/v3.0.1/liquid_feedback_core-v3.0.1.tar.gz
    tar -xvzf  liquid_feedback_core-v3.0.1.tar.gz
    cd liquid_feedback_core-v3.0.1
    make
    mkdir /opt/liquid_feedback_core
    cp core.sql lf_update /opt/liquid_feedback_core/
	chmod u+rwXs,g+rwXs -R /opt/liquid_feedback_core/
	chown www-data:www-data -R /opt/liquid_feedback_core/
    su - www-data
    cd /opt/liquid_feedback_core
    psql -U liquid -v ON_ERROR_STOP=1 -f core.sql lf
    psql -U liquid -d lf

    lf=> SELECT * FROM liquid_feedback_version;
     string | major | minor | revision 
    --------+-------+-------+----------
     3.0.1  | 3 	| 0 	| 1
    (1 row)

    lf=> SELECT * FROM system_setting;
     member_ttl 
    ------------
    (0 rows)

    lf=> INSERT INTO system_setting (member_ttl) VALUES ('1 year');
    INSERT 0 1

    lf=> SELECT * FROM contingent;
     polling | time_frame | text_entry_limit | initiative_limit 
    ---------+------------+------------------+------------------
    (0 rows)
    
    lf=> INSERT INTO contingent (polling, time_frame, text_entry_limit, initiative_limit) VALUES (false, '1 hour', 20, 6);
    INSERT 0 1

    lf=> INSERT INTO contingent (polling, time_frame, text_entry_limit, initiative_limit) VALUES (false, '1 day', 80, 12);
    INSERT 0 1

    lf=> INSERT INTO contingent (polling, time_frame, text_entry_limit, initiative_limit) VALUES (true, '1 hour', 200, 60);
    INSERT 0 1

    lf=> INSERT INTO contingent (polling, time_frame, text_entry_limit, initiative_limit) VALUES (true, '1 day', 800, 120);
    INSERT 0 1

    lf=> SELECT * FROM policy;
     id | index | active | name | description | polling | admission_time | discussion_time | verification_time | voting_time | issue_quorum_num | issue_quorum_den | initiative_quorum_num | initiative_quorum_den | direct_majority_num | direct_majority_den | direct_majority_strict | direct_majority_positive | direct_majority_non_negative | indirect_majority_num | indirect_majority_den | indirect_majority_strict | indirect_majority_positive | indirect_majority_non_negative | no_reverse_beat_path | no_multistage_majority 
    ----+-------+--------+------+-------------+---------+----------------+-----------------+-------------------+-------------+------------------+------------------+-----------------------+-----------------------+---------------------+---------------------+------------------------+--------------------------+------------------------------+-----------------------+-----------------------+--------------------------+----------------------------+--------------------------------+----------------------+------------------------
    (0 rows)
    
    lf=> INSERT INTO policy (index, name, admission_time, discussion_time, verification_time, voting_time, issue_quorum_num, issue_quorum_den, initiative_quorum_num, initiative_quorum_den) VALUES (1, 'Default policy', '8 days', '15 days', '8 days', '15 days', 10, 100, 10, 100);
    INSERT 0 1

    lf=> SELECT * FROM policy;
     id | index | active |  name  | description | polling | admission_time | discussion_time | verification_time | voting_time | issue_quorum_num | issue_quorum_den | initiative_quorum_num | initiative_quorum_den | direct_majority_num | direct_majority_den | direct_majority_strict | direct_majority_positive | direct_majority_non_negative | indirect_majority_num | indirect_majority_den | indirect_majority_strict | indirect_majority_positive | indirect_majority_non_negative | no_reverse_beat_path | no_multistage_majority 
    ----+-------+--------+----------------+-------------+---------+----------------+-----------------+-------------------+-------------+------------------+------------------+-----------------------+-----------------------+---------------------+---------------------+------------------------+--------------------------+------------------------------+-----------------------+-----------------------+--------------------------+----------------------------+--------------------------------+----------------------+------------------------
      1 | 1 | t  | Default policy | | f   | 8 days | 15 days | 8 days| 15 days |   10 |  100 |10 |   100 |   1 |   2 | t  |0 |0 | 1 | 2 | t|  0 |  0 | t| f
    (1 row)
    
    lf=> SELECT * FROM unit;
     id | parent_id | active | name | description | member_count | text_search_data 
    ----+-----------+--------+------+-------------+--------------+------------------
    (0 rows)
    
    lf=> INSERT INTO unit (name) VALUES ('Our organization');
    INSERT 0 1

    lf=> SELECT * FROM unit;
     id | parent_id | active |   name   | description | member_count | text_search_data 
    ----+-----------+--------+------------------+-------------+--------------+--------------------------
      1 |   | t  | Our organization | |  | 'organization':2 'our':1
    (1 row)
    
    lf=> SELECT * FROM area;
     id | unit_id | active | name | description | direct_member_count | member_weight | text_search_data 
    ----+---------+--------+------+-------------+---------------------+---------------+------------------
    (0 rows)
    
    lf=> INSERT INTO area (unit_id, name) VALUES (1, 'Default area');
    INSERT 0 1

    lf=> SELECT * FROM area;
     id | unit_id | active | name | description | direct_member_count | member_weight |   text_search_data   
    ----+---------+--------+--------------+-------------+---------------------+---------------+----------------------
      1 |   1 | t  | Default area | | |   | 'area':2 'default':1
    (1 row)
    
    lf=> SELECT * FROM allowed_policy;
     area_id | policy_id | default_policy 
    ---------+-----------+----------------
    (0 rows)
    
    lf=> INSERT INTO allowed_policy (area_id, policy_id, default_policy) VALUES (1, 1, TRUE);
    INSERT 0 1

    liquid_feedback=> \q

    exit
Install WebMCP:

    cd /root/install
    wget http://www.public-software-group.org/pub/projects/webmcp/v1.2.5/webmcp-v1.2.5.tar.gz
    tar -xvzf webmcp-v1.2.5.tar.gz
    cd webmcp-v1.2.5
    nano Makefile.options
	# append  -I /usr/include/lua5.1  at end of CFLAGS line
	# change -I /usr/include/postgresql -I /usr/include/postgresql/server to
	# -I /usr/include/postgresql/${version} -I /usr/include/postgresql/${version}/server
	# append  -I /usr/include/postgresql  at end of CFLAGS_PGSQL
    make
    mkdir /opt/webmcp
    cp -RL framework/* /opt/webmcp/
	chown www-data:www-data -R /opt/webmcp/
	chmod u+rwXs,g+rwXs -R /opt/webmcp/

Install RocketWiki LqFb-Edition:

    cd /root/install
    wget http://www.public-software-group.org/pub/projects/rocketwiki/liquid_feedback_edition/v0.4/rocketwiki-lqfb-v0.4.tar.gz
    tar -xvzf rocketwiki-lqfb-v0.4.tar.gz
    cd rocketwiki-lqfb-v0.4
    make
    mkdir /opt/rocketwiki-lqfb
    cp rocketwiki-lqfb rocketwiki-lqfb-compat /opt/rocketwiki-lqfb/
	chown www-data:www-data -R /opt/rocketwiki-lqfb/
	chmod u+rwXs,g+rwXs -R /opt/rocketwiki-lqfb/
Install LiquidFeedback-Frontend:

    cd /root/install
    wget http://www.public-software-group.org/pub/projects/liquid_feedback/frontend/v2.2.5/liquid_feedback_frontend-v2.2.5.tar.gz
    tar -xvzf liquid_feedback_frontend-v2.2.5.tar.gz
    mv liquid_feedback_frontend-v2.2.5 /opt/liquid_feedback_frontend
	chown www-data:www-data -R /opt/liquid_feedback_frontend
	chmod u+rwXs,g+rwXs -R /opt/liquid_feedback_frontend
Create HTML code for help texts:

    cd /opt/liquid_feedback_frontend/locale
    PATH=/opt/rocketwiki-lqfb:$PATH make
Make tmp directory of LiquidFeedback-Frontend writable for webserver:


    chown www-data /opt/liquid_feedback_frontend/tmp
Compile binary for fast delivery of member images:

    cd /opt/liquid_feedback_frontend/fastpath
    nano getpic.c
    check#define GETPIC_CONNINFO "dbname=lf"
    make

Configure mail system:


    dpkg-reconfigure exim4-config
Create webserver configuration for LiquidFeedback:

    cd /etc/lighttpd
    nano conf-available/60-liquidfeedback.conf
    server.modules += ("mod_cgi", "mod_rewrite", "mod_setenv")

Enable CGI-Execution of *.lua files through lua binary

    cgi.assign += ( ".lua" => "/usr/bin/lua5.1" )
    
    alias.url += ( "/lf/fastpath/" => "/opt/liquid_feedback_frontend/fastpath/",
    "/lf/static"=> "/opt/liquid_feedback_frontend/static",
    "/lf"   => "/opt/webmcp/cgi-bin" )

Configure environment for demo application

    $HTTP["url"] =~ "^/lf" {
      setenv.add-environment += (
    "LANG" => "en_US.UTF-8",
    "WEBMCP_APP_BASEPATH" => "/opt/liquid_feedback_frontend/",
    "WEBMCP_CONFIG_NAME"  => "myconfig")
    }

URL beautification

    url.rewrite-once += (
      # do not rewrite static URLs
      "^/lf/fastpath/(.*)$" => "/lf/fastpath/$1",
      "^/lf/static/(.*)$"   => "/lf/static/$1",
    
      # dynamic URLs
      "^/lf/([^\?]*)(\?(.*))?$" => "/lf/webmcp-wrapper.lua?_webmcp_path=$1&$3",    
    )
    
    $HTTP["url"] =~ "^/lf/fastpath/" {
      cgi.assign = ( "" => "" )
      setenv.add-response-header = ( "Cache-Control" => "private; max-age=86400" )
    }
    cd /etc/lighttpd/conf-enabled
    ln -s ../conf-available/60-liquidfeedback.conf .
Configure LiquidFeedback-Frontend:

    cd /opt/liquid_feedback_frontend/config
    cp example.lua myconfig.lua
    nano myconfig.lua
While configuring the LiquidFeedback-Frontend, use the following option to enable fast image loading:

    config.fastpath_url_func = function(member_id, image_type)
      return request.get_absolute_baseurl() .. "fastpath/getpic?" .. tostring(member_id) .. "+" .. tostring(image_type)
    end
Set database options

    config.database = { engine='postgresql', dbname='lf', user='liquid', password='feedback' }

Edit vHost name:


    config.absolute_base_url = "http://liquid-feedback.local/"

Execute lf_update once:

    su - www-data
    cd /opt/liquid_feedback_core
    ./lf_update dbname=lf user=liquid password=feedback && echo OK
    exit

The commands `lf_update dbname=lf user=liquid password=feedback` and `lf_update_suggestion_order dbname=lf user=liquid password=feedback` have to be executed regulary. Create the following shell script to call this command in an endless loop:


    nano /opt/liquid_feedback_core/lf_updated

    #!/bin/sh
    
    PIDFILE="/var/run/lf_updated.pid"
    PID=$$
    
    if [ -f "${PIDFILE}" ] && kill -CONT $( cat "${PIDFILE}" ); then
      echo "lf_updated is already running."
      exit 1
    fi
    
    echo "${PID}" "${PIDFILE}"
    
    while true; do
      su - www-data -c 'nice /opt/liquid_feedback_core/lf_update dbname=lf user=liquid password=feedback 2>&1 | logger -t "lf_updated"'
      su - www-data -c 'nice /opt/liquid_feedback_core/lf_update_suggestion_order dbname=lf user=liquid password=feedback 2>&1 | logger -t "lf_updated"'
      sleep 5
    done

    chmod +x /opt/liquid_feedback_core/lf_updated
Create the following init-script for lf_updated...

    nano /etc/init.d/lf_updated
    #! /bin/sh
    ### BEGIN INIT INFO
    # Provides:  lf_updated
    # Required-Start:$syslog
    # Required-Stop: $syslog
    # Default-Start: 2 3 4 5
    # Default-Stop:  0 1 6
    # Short-Description: lf_updated
    # Description:   Calls LiquidFeedback lf_update regulary
    ### END INIT INFO
    
    # PATH should only include /usr/* if it runs after the mountnfs.sh script
    PATH=/sbin:/usr/sbin:/bin:/usr/bin
    DESC="lf_updated"
    NAME=lf_updated
    DAEMON=/opt/liquid_feedback_core/lf_updated
    DAEMON_ARGS=""
    PIDFILE=/var/run/$NAME.pid
    SCRIPTNAME=/etc/init.d/$NAME
    
    . /lib/lsb/init-functions
    
    do_start()
    {
    	start-stop-daemon -b --start --quiet --pidfile $PIDFILE --exec $DAEMON --test > /dev/null || return 1
    	start-stop-daemon -b --start --quiet --pidfile $PIDFILE --exec $DAEMON -- $DAEMON_ARGS || return 2
    }
    
    do_stop()
    {
    	start-stop-daemon --stop --quiet --retry=TERM/30/KILL/5 --pidfile $PIDFILE --name $NAME
    	RETVAL="$?"
    	[ "$RETVAL" = 2 ] && return 2
    	start-stop-daemon --stop --quiet --oknodo --retry=0/30/KILL/5 --exec $DAEMON
    	[ "$?" = 2 ] && return 2
    	rm -f $PIDFILE
    	return "$RETVAL"
    }
    
    case "$1" in
      start)
    	[ "$VERBOSE" != no ] && log_daemon_msg "Starting $DESC" "$NAME"
    	do_start
    	case "$?" in
    		0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
    		2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    	esac
    	;;
      stop)
    	[ "$VERBOSE" != no ] && log_daemon_msg "Stopping $DESC" "$NAME"
    	do_stop
    	case "$?" in
    		0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
    		2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    	esac
    	;;
      status)
       	status_of_proc "$DAEMON" "$NAME" && exit 0 || exit $?
       	;;
      restart|force-reload)
    	log_daemon_msg "Restarting $DESC" "$NAME"
    	do_stop
    	case "$?" in
      		0|1)
    			do_start
    			case "$?" in
    				0) log_end_msg 0 ;;
    				1) log_end_msg 1 ;; # Old process is still running
    				*) log_end_msg 1 ;; # Failed to start
    			esac
    			;;
      		*)
    			# Failed to stop
    			log_end_msg 1
    			;;
    	esac
    	;;
      *)
    	echo "Usage: $SCRIPTNAME {start|stop|status|restart|force-reload}" >&2
    	exit 3
    	;;
	esac

and make the init script executable and let it start automatically at boot time

    chmod +x /etc/init.d/lf_updated
    update-rc.d lf_updated defaults
Start sending of event notifications:

     su - www-data -c 'cd /opt/liquid_feedback_frontend/ && nohup echo "Event:send_notifications_loop()" | ../webmcp/bin/webmcp_shell myconfig &>/dev/null &'
Restart the webserver:


    /etc/init.d/lighttpd restart
Create an administrator account:

    su - www-data
    cd /opt/liquid_feedback_core
    psql -U liquid -d lf
    
    lf=> INSERT INTO member (login, name, admin, invite_code) VALUES ('admin', 'Administrator', TRUE, _INSERT_ADMIN_INVITE_CODE_IN_SINGLE_QUOTES_HERE_);
    INSERT 0 1

    lf=> \q
    exit