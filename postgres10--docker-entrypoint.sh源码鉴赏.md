#### postgres10--docker-entrypoint.sh源码鉴赏

* 背景
  如上

* 说明

  postgres10的docker容器入口脚本docker-entrypoint.sh历经多年迭代，本身也在不断完善，总得来看应该有两个大的版本，笔者简称为旧版和新版。

  旧版链接：https://github.com/docker-library/postgres/blob/eff90effc6b5578be90bef93d96b3fceb1082a7c/docker-entrypoint.sh
  新版链接：https://github.com/docker-library/postgres/blob/master/docker-entrypoint.sh
  本文从旧版入手，然后再比较新旧两者的差异

* 正文
  ```bash
  #!/usr/bin/env bash
  set -Eeo pipefail
  # TODO swap to -Eeuo pipefail above (after handling all potentially-unset variables)
  # 加上-u参数 如果已经处理好所有潜在未设置的变量
  # set --help 查看-u参数的作用 如果变量未设置 也就是未初始化 那么视为报错 配合-e脚本则终止运行
  #      -u  Treat unset variables as an error when substituting.
  
  # 函数file_env()
  # 用法说明如下
  # usage: file_env VAR [DEFAULT]
  #    ie: file_env 'XYZ_DB_PASSWORD' 'example'
  # (will allow for "$XYZ_DB_PASSWORD_FILE" to fill in the value of
  #  "$XYZ_DB_PASSWORD" from a file, especially for Docker's secrets feature)
  file_env() {
  	local var="$1"
  	# 第一个位置参数给局部变量var
  	local fileVar="${var}_FILE"
  	# 配合第一个参数组合生成局部变量fileVar
  	local def="${2:-}"
  	# 将第二个位置参数给局部变量def 如果没有第二个变量那么直接将空给def 因为短横线后边为空
  	if [ "${!var:-}" ] && [ "${!fileVar:-}" ]; then
  	    # 两个变量都已设置 那么报错 退出 因为两个变量互斥
  		echo >&2 "error: both $var and $fileVar are set (but are exclusive)"
  		exit 1
  	fi
  	local val="$def"
  	# 将第二个位置参数生成的变量值def再复制给val 注意不是var
  	# 如果变量var已设置
  	if [ "${!var:-}" ]; then
  	    # 花括号扩展叹号 ${!var}
          # 取两次var的值 第一次取到的值本省又是一个变量名 再取一次值
  		val="${!var}"
      # 如果fileVar已设置
  	elif [ "${!fileVar:-}" ]; then
  	    # 这次取到的值再做一次标准输出 因为本身是个文件
  		val="$(< "${!fileVar}")"
  	fi
  	# 目前为止val要么是第二个参数的值 要么是var或fileVar中的一个
  	# 将局部变量声明为环境变量
  	export "$var"="$val"
  	# 取消fileVar变量
  	unset "$fileVar"
  }
  
  # 提取第一个位置参数的第0到1个字符 如果为短横线- 那么用set -- 初始化固定好所有位置参数
  if [ "${1:0:1}" = '-' ]; then
  	set -- postgres "$@"
  fi
  
  # 允许容器启动带--user参数 也就是自定义用户启动容器
  # allow the container to be started with `--user`
  # 如果第一个参数为postgres（上述set已经固定好第一个参数就是postgres）
  # 且id为0 也就是root用户 那么创建相应目录及设置对应权限
  if [ "$1" = 'postgres' ] && [ "$(id -u)" = '0' ]; then
  	mkdir -p "$PGDATA"
  	chown -R postgres "$PGDATA"
  	chmod 700 "$PGDATA"
  
  	mkdir -p /var/run/postgresql
  	chown -R postgres /var/run/postgresql
  	chmod 775 /var/run/postgresql
  
  	# Create the transaction log directory before initdb is run (below) so the directory is owned by the correct user
  	# 如果变量POSTGRES_INITDB_WALDIR已经存在且初始化
  	if [ "$POSTGRES_INITDB_WALDIR" ]; then
  		mkdir -p "$POSTGRES_INITDB_WALDIR"
  		chown -R postgres "$POSTGRES_INITDB_WALDIR"
  		chmod 700 "$POSTGRES_INITDB_WALDIR"
  	fi
  
      # exec进程替换 且用gosu提升权限来运行
      # 注意postgres后边 在所有位置参数$@前 又添加了一个BASH_SOURCE
  	exec gosu postgres "$BASH_SOURCE" "$@"
  fi
  
  # 如果第一个参数是postgres（不关后边是不是root用户）都将PGDATA目录权限给当前用户
  # 2>/dev/null || :
  # 上述用法的唯一目的就是以成功状态退出 也就是0状态退出
  # 这跟开头的set -e有关联 作者不想因为这里的意外而直接退出脚本
  # 冒号单独使用时表示空命令 什么也不做 所以返回状态为0
  # https://superuser.com/questions/1022374/what-does-mean-in-the-context-of-a-shell-script
  if [ "$1" = 'postgres' ]; then
  	mkdir -p "$PGDATA"
  	chown -R "$(id -u)" "$PGDATA" 2>/dev/null || :
  	chmod 700 "$PGDATA" 2>/dev/null || :
  
  	# look specifically for PG_VERSION, as it is expected in the DB dir
  	# if -s 文件存在且不为空 叹号! 取反； 也就是文件为空则。。
  	if [ ! -s "$PGDATA/PG_VERSION" ]; then
  		# "initdb" is particular about the current user existing in "/etc/passwd", so we use "nss_wrapper" to fake that if necessary
  		# see https://github.com/docker-library/postgres/pull/253, https://github.com/docker-library/postgres/issues/359, https://cwrap.org/nss_wrapper.html
  		# getent 获取当前登陆用户的信息
  		if ! getent passwd "$(id -u)" &> /dev/null && [ -e /usr/lib/libnss_wrapper.so ]; then
  			export LD_PRELOAD='/usr/lib/libnss_wrapper.so'
  			export NSS_WRAPPER_PASSWD="$(mktemp)"
  			export NSS_WRAPPER_GROUP="$(mktemp)"
  			echo "postgres:x:$(id -u):$(id -g):PostgreSQL:$PGDATA:/bin/false" > "$NSS_WRAPPER_PASSWD"
  			echo "postgres:x:$(id -g):" > "$NSS_WRAPPER_GROUP"
  		fi
  
  		file_env 'POSTGRES_INITDB_ARGS'
  		if [ "$POSTGRES_INITDB_WALDIR" ]; then
  			export POSTGRES_INITDB_ARGS="$POSTGRES_INITDB_ARGS --waldir $POSTGRES_INITDB_WALDIR"
  		fi
  		eval "initdb --username=postgres $POSTGRES_INITDB_ARGS"
  
  		# unset/cleanup "nss_wrapper" bits
  		if [ "${LD_PRELOAD:-}" = '/usr/lib/libnss_wrapper.so' ]; then
  			rm -f "$NSS_WRAPPER_PASSWD" "$NSS_WRAPPER_GROUP"
  			unset LD_PRELOAD NSS_WRAPPER_PASSWD NSS_WRAPPER_GROUP
  		fi
  
  		# check password first so we can output the warning before postgres
  		# messes it up
  		file_env 'POSTGRES_PASSWORD'
  		if [ "$POSTGRES_PASSWORD" ]; then
  			pass="PASSWORD '$POSTGRES_PASSWORD'"
  			authMethod=md5
  		else
  		    # tab键会被缩进 但是空格不会 因此下面格式会被保留
  			# The - option suppresses leading tabs but *not* spaces. :)
  			cat >&2 <<-'EOWARN'
  				****************************************************
  				WARNING: No password has been set for the database.
  				         This will allow anyone with access to the
  				         Postgres port to access your database. In
  				         Docker's default configuration, this is
  				         effectively any other container on the same
  				         system.
  
  				         Use "-e POSTGRES_PASSWORD=password" to set
  				         it in "docker run".
  				****************************************************
  			EOWARN
  
  			pass=
  			authMethod=trust
  		fi
  
  		{
  			echo
  			echo "host all all all $authMethod"
  		} >> "$PGDATA/pg_hba.conf"
  		# 使用花括号 配合echo往文件里追加多行内容 也是绝了
  		# 花括号会把多个echo的输出看做一个整体 然后追加到文件
  
  		# internal start of server in order to allow set-up using psql-client
  		# does not listen on external TCP/IP and waits until start finishes
  		PGUSER="${PGUSER:-postgres}" \
  		pg_ctl -D "$PGDATA" \
  			-o "-c listen_addresses=''" \
  			-w start
  
  		file_env 'POSTGRES_USER' 'postgres'
  		file_env 'POSTGRES_DB' "$POSTGRES_USER"
  
  		psql=( psql -v ON_ERROR_STOP=1 )
  		# 注意这里是把后边的命令组合转换成一个数组 而不是简单的字符串替换
  		# 所以后边引用的使用用${psql[@]} 也就是数组里的所有元素
  
  		if [ "$POSTGRES_DB" != 'postgres' ]; then
  			"${psql[@]}" --username postgres <<-EOSQL
  				CREATE DATABASE "$POSTGRES_DB" ;
  			EOSQL
  			echo
  		fi
  
  		if [ "$POSTGRES_USER" = 'postgres' ]; then
  			op='ALTER'
  		else
  			op='CREATE'
  		fi
  		"${psql[@]}" --username postgres <<-EOSQL
  			$op USER "$POSTGRES_USER" WITH SUPERUSER $pass ;
  		EOSQL
  		echo
  
          # 数组追加 因为psql本身是个数组 再追加后边圆括号的元素到数组
  		psql+=( --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" )
  
  		echo
  		# 遍历初始化目录里的文件 根据类型依次执行初始化
  		for f in /docker-entrypoint-initdb.d/*; do
  			case "$f" in
  				*.sh)
  					# https://github.com/docker-library/postgres/issues/450#issuecomment-393167936
  					# https://github.com/docker-library/postgres/pull/452
  					if [ -x "$f" ]; then
  						echo "$0: running $f"
  						"$f"
  					else
  						echo "$0: sourcing $f"
  						. "$f"
  					fi
  					;;
  				*.sql)    echo "$0: running $f"; "${psql[@]}" -f "$f"; echo ;;
  				*.sql.gz) echo "$0: running $f"; gunzip -c "$f" | "${psql[@]}"; echo ;;
  				*)        echo "$0: ignoring $f" ;;
  			esac
  			echo
  		done
  
  		PGUSER="${PGUSER:-postgres}" \
  		pg_ctl -D "$PGDATA" -m fast -w stop
  
  		echo
  		echo 'PostgreSQL init process complete; ready for start up.'
  		echo
  	fi
  fi
  
  # exec命令替换所有的位置参数
  exec "$@"
  ```

* 新版改动之多了一个_is_sourced()函数

  ```
  # 如果函数的参数大于等于2 且函数的第一个参数为_is_sourced 且第二个参数为source
  # 方括号里的一个等号和两个等号都一样 表示比较等于
  # 三个条件都为真表示_is_sourced()函数被重复执行
  # check to see if this file is being run or sourced from another script
  _is_sourced() {
  	# https://unix.stackexchange.com/a/215279
  	[ "${#FUNCNAME[@]}" -ge 2 ] \
  		&& [ "${FUNCNAME[0]}" = '_is_sourced' ] \
  		&& [ "${FUNCNAME[1]}" = 'source' ]
  }
  
  ```

* 新版源码鉴赏
  ```bash
  #!/usr/bin/env bash
  set -Eeo pipefail
  # TODO swap to -Eeuo pipefail above (after handling all potentially-unset variables)
  
  # usage: file_env VAR [DEFAULT]
  #    ie: file_env 'XYZ_DB_PASSWORD' 'example'
  # (will allow for "$XYZ_DB_PASSWORD_FILE" to fill in the value of
  #  "$XYZ_DB_PASSWORD" from a file, especially for Docker's secrets feature)
  file_env() {
  	local var="$1"
  	local fileVar="${var}_FILE"
  	local def="${2:-}"
  	if [ "${!var:-}" ] && [ "${!fileVar:-}" ]; then
  		printf >&2 'error: both %s and %s are set (but are exclusive)\n' "$var" "$fileVar"
  		exit 1
  	fi
  	local val="$def"
  	if [ "${!var:-}" ]; then
  		val="${!var}"
  	elif [ "${!fileVar:-}" ]; then
  		val="$(< "${!fileVar}")"
  	fi
  	export "$var"="$val"
  	unset "$fileVar"
  }
  
  # check to see if this file is being run or sourced from another script
  _is_sourced() {
  	# https://unix.stackexchange.com/a/215279
  	[ "${#FUNCNAME[@]}" -ge 2 ] \
  		&& [ "${FUNCNAME[0]}" = '_is_sourced' ] \
  		&& [ "${FUNCNAME[1]}" = 'source' ]
  }
  
  # 等同于旧版以root身份初始化
  # used to create initial postgres directories and if run as root, ensure ownership to the "postgres" user
  docker_create_db_directories() {
  	local user; user="$(id -u)"
  
  	mkdir -p "$PGDATA"
  	# ignore failure since there are cases where we can't chmod (and PostgreSQL might fail later anyhow - it's picky about permissions of this directory)
  	# 新版明确指出chmod可能会出现failure 因此加了与空命令冒号逻辑或 保证返回值为0
  	chmod 00700 "$PGDATA" || :
  
  	# ignore failure since it will be fine when using the image provided directory; see also https://github.com/docker-library/postgres/pull/289
  	mkdir -p /var/run/postgresql || :
  	chmod 03775 /var/run/postgresql || :
  
  	# Create the transaction log directory before initdb is run so the directory is owned by the correct user
  	if [ -n "${POSTGRES_INITDB_WALDIR:-}" ]; then
  		mkdir -p "$POSTGRES_INITDB_WALDIR"
  		if [ "$user" = '0' ]; then
  			find "$POSTGRES_INITDB_WALDIR" \! -user postgres -exec chown postgres '{}' +
  		fi
  		chmod 700 "$POSTGRES_INITDB_WALDIR"
  	fi
  
  	# allow the container to be started with `--user`
  	if [ "$user" = '0' ]; then
  		find "$PGDATA" \! -user postgres -exec chown postgres '{}' +
  		find /var/run/postgresql \! -user postgres -exec chown postgres '{}' +
  	fi
  }
  
  # 等同于旧版以非root身份初始化
  # initialize empty PGDATA directory with new database via 'initdb'
  # arguments to `initdb` can be passed via POSTGRES_INITDB_ARGS or as arguments to this function
  # `initdb` automatically creates the "postgres", "template0", and "template1" dbnames
  # this is also where the database user is created, specified by `POSTGRES_USER` env
  docker_init_database_dir() {
  	# "initdb" is particular about the current user existing in "/etc/passwd", so we use "nss_wrapper" to fake that if necessary
  	# see https://github.com/docker-library/postgres/pull/253, https://github.com/docker-library/postgres/issues/359, https://cwrap.org/nss_wrapper.html
  	local uid; uid="$(id -u)"
  	if ! getent passwd "$uid" &> /dev/null; then
  		# see if we can find a suitable "libnss_wrapper.so" (https://salsa.debian.org/sssd-team/nss-wrapper/-/commit/b9925a653a54e24d09d9b498a2d913729f7abb15)
  		local wrapper
  		for wrapper in {/usr,}/lib{/*,}/libnss_wrapper.so; do
  			if [ -s "$wrapper" ]; then
  				NSS_WRAPPER_PASSWD="$(mktemp)"
  				NSS_WRAPPER_GROUP="$(mktemp)"
  				export LD_PRELOAD="$wrapper" NSS_WRAPPER_PASSWD NSS_WRAPPER_GROUP
  				local gid; gid="$(id -g)"
  				printf 'postgres:x:%s:%s:PostgreSQL:%s:/bin/false\n' "$uid" "$gid" "$PGDATA" > "$NSS_WRAPPER_PASSWD"
  				printf 'postgres:x:%s:\n' "$gid" > "$NSS_WRAPPER_GROUP"
  				break
  			fi
  		done
  	fi
  
  	if [ -n "${POSTGRES_INITDB_WALDIR:-}" ]; then
  		set -- --waldir "$POSTGRES_INITDB_WALDIR" "$@"
  	fi
  
  	# --pwfile refuses to handle a properly-empty file (hence the "\n"): https://github.com/docker-library/postgres/issues/1025
  	eval 'initdb --username="$POSTGRES_USER" --pwfile=<(printf "%s\n" "$POSTGRES_PASSWORD") '"$POSTGRES_INITDB_ARGS"' "$@"'
  
  	# unset/cleanup "nss_wrapper" bits
  	if [[ "${LD_PRELOAD:-}" == */libnss_wrapper.so ]]; then
  		rm -f "$NSS_WRAPPER_PASSWD" "$NSS_WRAPPER_GROUP"
  		unset LD_PRELOAD NSS_WRAPPER_PASSWD NSS_WRAPPER_GROUP
  	fi
  }
  
  # LARGE WARNING 如果出现以下情况
  # print large warning if POSTGRES_PASSWORD is long
  # error if both POSTGRES_PASSWORD is empty and POSTGRES_HOST_AUTH_METHOD is not 'trust'
  # print large warning if POSTGRES_HOST_AUTH_METHOD is set to 'trust'
  # assumes database is not set up, ie: [ -z "$DATABASE_ALREADY_EXISTS" ]
  docker_verify_minimum_env() {
  	# check password first so we can output the warning before postgres
  	# messes it up
  	if [ "${#POSTGRES_PASSWORD}" -ge 100 ]; then
  		cat >&2 <<-'EOWARN'
  
  			WARNING: The supplied POSTGRES_PASSWORD is 100+ characters.
  
  			  This will not work if used via PGPASSWORD with "psql".
  
  			  https://www.postgresql.org/message-id/flat/E1Rqxp2-0004Qt-PL%40wrigleys.postgresql.org (BUG #6412)
  			  https://github.com/docker-library/postgres/issues/507
  
  		EOWARN
  	fi
  	if [ -z "$POSTGRES_PASSWORD" ] && [ 'trust' != "$POSTGRES_HOST_AUTH_METHOD" ]; then
  		# The - option suppresses leading tabs but *not* spaces. :)
  		cat >&2 <<-'EOE'
  			Error: Database is uninitialized and superuser password is not specified.
  			       You must specify POSTGRES_PASSWORD to a non-empty value for the
  			       superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".
  
  			       You may also use "POSTGRES_HOST_AUTH_METHOD=trust" to allow all
  			       connections without a password. This is *not* recommended.
  
  			       See PostgreSQL documentation about "trust":
  			       https://www.postgresql.org/docs/current/auth-trust.html
  		EOE
  		exit 1
  	fi
  	if [ 'trust' = "$POSTGRES_HOST_AUTH_METHOD" ]; then
  		cat >&2 <<-'EOWARN'
  			********************************************************************************
  			WARNING: POSTGRES_HOST_AUTH_METHOD has been set to "trust". This will allow
  			         anyone with access to the Postgres port to access your database without
  			         a password, even if POSTGRES_PASSWORD is set. See PostgreSQL
  			         documentation about "trust":
  			         https://www.postgresql.org/docs/current/auth-trust.html
  			         In Docker's default configuration, this is effectively any other
  			         container on the same system.
  
  			         It is not recommended to use POSTGRES_HOST_AUTH_METHOD=trust. Replace
  			         it with "-e POSTGRES_PASSWORD=password" instead to set a password in
  			         "docker run".
  			********************************************************************************
  		EOWARN
  	fi
  }
  
  # 遍历初始化文件进行初始化
  # usage: docker_process_init_files [file [file [...]]]
  #    ie: docker_process_init_files /always-initdb.d/*
  # process initializer files, based on file extensions and permissions
  docker_process_init_files() {
  	# psql here for backwards compatibility "${psql[@]}"
      # psql 这里用于向后兼容性 “${psql[@]}”
  	psql=( docker_process_sql )
  
  	printf '\n'
  	local f
  	for f; do
  		case "$f" in
  			*.sh)
  				# https://github.com/docker-library/postgres/issues/450#issuecomment-393167936
  				# https://github.com/docker-library/postgres/pull/452
  				if [ -x "$f" ]; then
  					printf '%s: running %s\n' "$0" "$f"
  					"$f"
  				else
  					printf '%s: sourcing %s\n' "$0" "$f"
  					. "$f"
  				fi
  				;;
  			*.sql)     printf '%s: running %s\n' "$0" "$f"; docker_process_sql -f "$f"; printf '\n' ;;
  			*.sql.gz)  printf '%s: running %s\n' "$0" "$f"; gunzip -c "$f" | docker_process_sql; printf '\n' ;;
  			*.sql.xz)  printf '%s: running %s\n' "$0" "$f"; xzcat "$f" | docker_process_sql; printf '\n' ;;
  			*.sql.zst) printf '%s: running %s\n' "$0" "$f"; zstd -dc "$f" | docker_process_sql; printf '\n' ;;
  			*)         printf '%s: ignoring %s\n' "$0" "$f" ;;
  		esac
  		printf '\n'
  	done
  }
  
  # Execute sql script, passed via stdin (or -f flag of pqsl)
  # usage: docker_process_sql [psql-cli-args]
  #    ie: docker_process_sql --dbname=mydb <<<'INSERT ...'
  #    ie: docker_process_sql -f my-file.sql
  #    ie: docker_process_sql <my-file.sql
  docker_process_sql() {
  	local query_runner=( psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --no-password --no-psqlrc )
  	if [ -n "$POSTGRES_DB" ]; then
  		query_runner+=( --dbname "$POSTGRES_DB" )
  	fi
  
      # 这个地方不太好理解 为啥子把他们写在一行 不过当我想到下面的用法时我就豁然开朗了
      # PGPASSWORD=xxx psql ----
      # psql 或者 pg_dump[all] 带着PGPASSWORD查询或备份时 这样不用再次输入密码
      # 下面命令用法类似 只不过变量换成了PGHOST PGHOSTADDR
  	PGHOST= PGHOSTADDR= "${query_runner[@]}" "$@"
  }
  
  # create initial database
  # uses environment variables for input: POSTGRES_DB
  docker_setup_db() {
  	local dbAlreadyExists
  	dbAlreadyExists="$(
  		POSTGRES_DB= docker_process_sql --dbname postgres --set db="$POSTGRES_DB" --tuples-only <<-'EOSQL'
  			SELECT 1 FROM pg_database WHERE datname = :'db' ;
  			# 这里的sql语法中的冒号用到了字典 也可以叫锚点 db是字典key db变量的值为字典value
  			# 如果db的值为mario等效于SELECT 1 FROM pg_database WHERE datname = 'mario' ;
  		EOSQL
  	)"
  	if [ -z "$dbAlreadyExists" ]; then
  		POSTGRES_DB= docker_process_sql --dbname postgres --set db="$POSTGRES_DB" <<-'EOSQL'
  			CREATE DATABASE :"db" ;
  			# 这里的锚点用的双引号 不像前边的单引号
  		EOSQL
  		printf '\n'
  	fi
  }
  
  # Loads various settings that are used elsewhere in the script
  # This should be called before any other functions
  docker_setup_env() {
  	file_env 'POSTGRES_PASSWORD'
  
  	file_env 'POSTGRES_USER' 'postgres'
  	file_env 'POSTGRES_DB' "$POSTGRES_USER"
  	file_env 'POSTGRES_INITDB_ARGS'
  	: "${POSTGRES_HOST_AUTH_METHOD:=}"
  	# 这里的冒号 类似于井号 做注释用
  
  	declare -g DATABASE_ALREADY_EXISTS
  	# look specifically for PG_VERSION, as it is expected in the DB dir
  	if [ -s "$PGDATA/PG_VERSION" ]; then
  		DATABASE_ALREADY_EXISTS='true'
  	fi
  }
  
  # append POSTGRES_HOST_AUTH_METHOD to pg_hba.conf for "host" connections
  # all arguments will be passed along as arguments to `postgres` for getting the value of 'password_encryption'
  pg_setup_hba_conf() {
  	# default authentication method is md5 on versions before 14
  	# https://www.postgresql.org/about/news/postgresql-14-released-2318/
  	if [ "$1" = 'postgres' ]; then
  		shift
  	fi
  	local auth
  	# check the default/configured encryption and use that as the auth method
  	auth="$(postgres -C password_encryption "$@")"
  	: "${POSTGRES_HOST_AUTH_METHOD:=$auth}"
  	{
  		printf '\n'
  		if [ 'trust' = "$POSTGRES_HOST_AUTH_METHOD" ]; then
  			printf '# warning trust is enabled for all connections\n'
  			printf '# see https://www.postgresql.org/docs/12/auth-trust.html\n'
  		fi
  		printf 'host all all all %s\n' "$POSTGRES_HOST_AUTH_METHOD"
  	} >> "$PGDATA/pg_hba.conf"
  }
  
  # start socket-only postgresql server for setting up or running scripts
  # all arguments will be passed along as arguments to `postgres` (via pg_ctl)
  docker_temp_server_start() {
  	if [ "$1" = 'postgres' ]; then
  		shift
  	fi
  
  	# internal start of server in order to allow setup using psql client
  	# does not listen on external TCP/IP and waits until start finishes
  	set -- "$@" -c listen_addresses='' -p "${PGPORT:-5432}"
  
  	PGUSER="${PGUSER:-$POSTGRES_USER}" \
  	pg_ctl -D "$PGDATA" \
  		-o "$(printf '%q ' "$@")" \
  		-w start
  }
  
  # stop postgresql server after done setting up user and running scripts
  docker_temp_server_stop() {
  	PGUSER="${PGUSER:-postgres}" \
  	pg_ctl -D "$PGDATA" -m fast -w stop
  }
  
  # check arguments for an option that would cause postgres to stop
  # return true if there is one
  _pg_want_help() {
  	local arg
  	for arg; do
  		case "$arg" in
  			# postgres --help | grep 'then exit'
  			# leaving out -C on purpose since it always fails and is unhelpful:
  			# postgres: could not access the server configuration file "/var/lib/postgresql/data/postgresql.conf": No such file or directory
  			-'?'|--help|--describe-config|-V|--version)
  				return 0
  				;;
  		esac
  	done
  	return 1
  }
  
  _main() {
  	# 如果第一个 arg 看起来像一个标志，假设我们要运行 postgres 服务器
  	# if first arg looks like a flag, assume we want to run postgres server
  	if [ "${1:0:1}" = '-' ]; then
  		set -- postgres "$@"
  	fi
  
  	if [ "$1" = 'postgres' ] && ! _pg_want_help "$@"; then
  	# 如果第一个参数为postgres 且其余参数没有需要帮助的信息
  		docker_setup_env
  		# setup data directories and permissions (when run as root)
  		docker_create_db_directories
  		if [ "$(id -u)" = '0' ]; then
  			# then restart script as postgres user
  			exec gosu postgres "$BASH_SOURCE" "$@"
  		fi
  
  		# only run initialization on an empty data directory
  		# 新挂载的磁盘带“lost+found”目录也不行
  		# “lost+found”目录用于存放操作系统ext文件系统意外产生的数据
  		if [ -z "$DATABASE_ALREADY_EXISTS" ]; then
  			docker_verify_minimum_env
  
  			# check dir permissions to reduce likelihood of half-initialized database
  			ls /docker-entrypoint-initdb.d/ > /dev/null
  
  			docker_init_database_dir
  			pg_setup_hba_conf "$@"
  
  			# PGPASSWORD is required for psql when authentication is required for 'local' connections via pg_hba.conf and is otherwise harmless
  			# e.g. when '--auth=md5' or '--auth-local=md5' is used in POSTGRES_INITDB_ARGS
  			export PGPASSWORD="${PGPASSWORD:-$POSTGRES_PASSWORD}"
  			docker_temp_server_start "$@"
  
  			docker_setup_db
  			docker_process_init_files /docker-entrypoint-initdb.d/*
  
  			docker_temp_server_stop
  			unset PGPASSWORD
  
  			cat <<-'EOM'
  
  				PostgreSQL init process complete; ready for start up.
  
  			EOM
  		else
  			cat <<-'EOM'
  
  				PostgreSQL Database directory appears to contain a database; Skipping initialization
  
  			EOM
  		fi
  	fi
  
  	exec "$@"
  }
  
  if ! _is_sourced; then
  	_main "$@"
  fi
  ```

* 附初始化日志

  ```
  + _is_sourced
  + '[' 2 -ge 2 ']'
  + '[' _is_sourced = _is_sourced ']'
  + '[' main = source ']'
  + _main -c max_connections=5000 -c shared_buffers=8GB
  + '[' - = - ']'
  + set -- postgres -c max_connections=5000 -c shared_buffers=8GB
  + '[' postgres = postgres ']'
  + _pg_want_help postgres -c max_connections=5000 -c shared_buffers=8GB
  + local arg
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + return 1
  + docker_setup_env
  + file_env POSTGRES_PASSWORD
  + local var=POSTGRES_PASSWORD
  + local fileVar=POSTGRES_PASSWORD_FILE
  + local def=
  + '[' 'Qsa97234LfregAhh33' ']'
  + '[' '' ']'
  + local val=
  + '[' 'Qsa97234LfregAhh33' ']'
  + val='Qsa97234LfregAhh33'
  + export 'POSTGRES_PASSWORD=Qsa97234LfregAhh33'
  + POSTGRES_PASSWORD='Qsa97234LfregAhh33'
  + unset POSTGRES_PASSWORD_FILE
  + file_env POSTGRES_USER postgres
  + local var=POSTGRES_USER
  + local fileVar=POSTGRES_USER_FILE
  + local def=postgres
  + '[' '' ']'
  + local val=postgres
  + '[' '' ']'
  + '[' '' ']'
  + export POSTGRES_USER=postgres
  + POSTGRES_USER=postgres
  + unset POSTGRES_USER_FILE
  + file_env POSTGRES_DB postgres
  + local var=POSTGRES_DB
  + local fileVar=POSTGRES_DB_FILE
  + local def=postgres
  + '[' igix ']'
  + '[' '' ']'
  + local val=postgres
  + '[' igix ']'
  + val=igix
  + export POSTGRES_DB=igix
  + POSTGRES_DB=igix
  + unset POSTGRES_DB_FILE
  + file_env POSTGRES_INITDB_ARGS
  + local var=POSTGRES_INITDB_ARGS
  + local fileVar=POSTGRES_INITDB_ARGS_FILE
  + local def=
  + '[' '' ']'
  + local val=
  + '[' '' ']'
  + '[' '' ']'
  + export POSTGRES_INITDB_ARGS=
  + POSTGRES_INITDB_ARGS=
  + unset POSTGRES_INITDB_ARGS_FILE
  + : ''
  + declare -g DATABASE_ALREADY_EXISTS
  + '[' -s /var/lib/postgresql/data/PG_VERSION ']'
  + docker_create_db_directories
  + local user
  ++ id -u
  + user=0
  + mkdir -p /var/lib/postgresql/data
  + chmod 700 /var/lib/postgresql/data
  + mkdir -p /var/run/postgresql
  + chmod 775 /var/run/postgresql
  + '[' -n '' ']'
  + '[' 0 = 0 ']'
  + find /var/lib/postgresql/data '!' -user postgres -exec chown postgres '{}' +
  + find /var/run/postgresql '!' -user postgres -exec chown postgres '{}' +
  ++ id -u
  + '[' 0 = 0 ']'
  + exec gosu postgres /usr/local/bin/docker-entrypoint.sh postgres -c max_connections=5000 -c shared_buffers=8GB
  + _is_sourced
  + '[' 2 -ge 2 ']'
  + '[' _is_sourced = _is_sourced ']'
  + '[' main = source ']'
  + _main postgres -c max_connections=5000 -c shared_buffers=8GB
  + '[' p = - ']'
  + '[' postgres = postgres ']'
  + _pg_want_help postgres -c max_connections=5000 -c shared_buffers=8GB
  + local arg
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + for arg in "$@"
  + case "$arg" in
  + return 1
  + docker_setup_env
  + file_env POSTGRES_PASSWORD
  + local var=POSTGRES_PASSWORD
  + local fileVar=POSTGRES_PASSWORD_FILE
  + local def=
  + '[' 'Qsa97234LfregAhh33' ']'
  + '[' '' ']'
  + local val=
  + '[' 'Qsa97234LfregAhh33' ']'
  + val='Qsa97234LfregAhh33'
  + export 'POSTGRES_PASSWORD=Qsa97234LfregAhh33'
  + POSTGRES_PASSWORD='Qsa97234LfregAhh33'
  + unset POSTGRES_PASSWORD_FILE
  + file_env POSTGRES_USER postgres
  + local var=POSTGRES_USER
  + local fileVar=POSTGRES_USER_FILE
  + local def=postgres
  + '[' postgres ']'
  + '[' '' ']'
  + local val=postgres
  + '[' postgres ']'
  + val=postgres
  + export POSTGRES_USER=postgres
  + POSTGRES_USER=postgres
  + unset POSTGRES_USER_FILE
  + file_env POSTGRES_DB postgres
  + local var=POSTGRES_DB
  + local fileVar=POSTGRES_DB_FILE
  + local def=postgres
  + '[' igix ']'
  + '[' '' ']'
  + local val=postgres
  + '[' igix ']'
  + val=igix
  + export POSTGRES_DB=igix
  + POSTGRES_DB=igix
  + unset POSTGRES_DB_FILE
  + file_env POSTGRES_INITDB_ARGS
  + local var=POSTGRES_INITDB_ARGS
  + local fileVar=POSTGRES_INITDB_ARGS_FILE
  + local def=
  + '[' '' ']'
  + local val=
  + '[' '' ']'
  + '[' '' ']'
  + export POSTGRES_INITDB_ARGS=
  + POSTGRES_INITDB_ARGS=
  + unset POSTGRES_INITDB_ARGS_FILE
  + : ''
  + declare -g DATABASE_ALREADY_EXISTS
  + '[' -s /var/lib/postgresql/data/PG_VERSION ']'
  + docker_create_db_directories
  + local user
  ++ id -u
  + user=999
  + mkdir -p /var/lib/postgresql/data
  + chmod 700 /var/lib/postgresql/data
  + mkdir -p /var/run/postgresql
  + chmod 775 /var/run/postgresql
  + '[' -n '' ']'
  + '[' 999 = 0 ']'
  ++ id -u
  + '[' 999 = 0 ']'
  + '[' -z '' ']'
  + docker_verify_minimum_env
  + '[' 16 -ge 100 ']'
  + '[' -z 'Qsa97234LfregAhh33' ']'
  + '[' trust = '' ']'
  + ls /docker-entrypoint-initdb.d/
  + docker_init_database_dir
  + local uid
  ++ id -u
  + uid=999
  + getent passwd 999
  + '[' -n '' ']'
  + eval 'initdb --username="$POSTGRES_USER" --pwfile=<(echo "$POSTGRES_PASSWORD")  "$@"'
  ++ initdb --username=postgres --pwfile=/dev/fd/63
  +++ echo 'Qsa97234LfregAhh33'
  The files belonging to this database system will be owned by user "postgres".
  This user must also own the server process.
  
  The database cluster will be initialized with locale "en_US.utf8".
  The default database encoding has accordingly been set to "UTF8".
  The default text search configuration will be set to "english".
  
  Data page checksums are disabled.
  
  fixing permissions on existing directory /var/lib/postgresql/data ... ok
  creating subdirectories ... ok
  selecting default max_connections ... 100
  selecting default shared_buffers ... 128MB
  selecting default timezone ... Etc/UTC
  selecting dynamic shared memory implementation ... posix
  creating configuration files ... ok
  running bootstrap script ... ok
  performing post-bootstrap initialization ... ok
  
  WARNING: enabling "trust" authentication for local connections
  You can change this by editing pg_hba.conf or using the option -A, or
  --auth-local and --auth-host, the next time you run initdb.
  syncing data to disk ... ok
  
  Success. You can now start the database server using:
  
      pg_ctl -D /var/lib/postgresql/data -l logfile start
  
  + [[ '' == */libnss_wrapper.so ]]
  + pg_setup_hba_conf postgres -c max_connections=5000 -c shared_buffers=8GB
  + '[' postgres = postgres ']'
  + shift
  + local auth
  ++ postgres -C password_encryption -c max_connections=5000 -c shared_buffers=8GB
  + auth=md5
  + : md5
  + echo
  + '[' trust = md5 ']'
  + echo 'host all all all md5'
  + export 'PGPASSWORD=Qsa97234LfregAhh33'
  + PGPASSWORD='Qsa97234LfregAhh33'
  + docker_temp_server_start postgres -c max_connections=5000 -c shared_buffers=8GB
  + '[' postgres = postgres ']'
  + shift
  + set -- -c max_connections=5000 -c shared_buffers=8GB -c listen_addresses= -p 5432
  ++ printf '%q ' -c max_connections=5000 -c shared_buffers=8GB -c listen_addresses= -p 5432
  + PGUSER=postgres
  + pg_ctl -D /var/lib/postgresql/data -o '-c max_connections=5000 -c shared_buffers=8GB -c listen_addresses= -p 5432 ' -w start
  waiting for server to start....2023-09-15 07:29:28.681 UTC [48] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
  2023-09-15 07:29:28.853 UTC [49] LOG:  database system was shut down at 2023-09-15 07:29:28 UTC
  2023-09-15 07:29:28.860 UTC [48] LOG:  database system is ready to accept connections
   done
  server started
  + docker_setup_db
  + local dbAlreadyExists
  ++ POSTGRES_DB=
  ++ docker_process_sql --dbname postgres --set db=igix --tuples-only
  ++ query_runner=(psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --no-password --no-psqlrc)
  ++ local query_runner
  ++ '[' -n '' ']'
  ++ PGHOST=
  ++ PGHOSTADDR=
  ++ psql -v ON_ERROR_STOP=1 --username postgres --no-password --no-psqlrc --dbname postgres --set db=igix --tuples-only
  + dbAlreadyExists=
  + '[' -z '' ']'
  + POSTGRES_DB=
  + docker_process_sql --dbname postgres --set db=igix
  + query_runner=(psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --no-password --no-psqlrc)
  + local query_runner
  + '[' -n '' ']'
  + PGHOST=
  + PGHOSTADDR=
  + psql -v ON_ERROR_STOP=1 --username postgres --no-password --no-psqlrc --dbname postgres --set db=igix
  CREATE DATABASE
  
  + echo
  + docker_process_init_files /docker-entrypoint-initdb.d/001-init-user-db.sh
  + psql=(docker_process_sql)
  
  + echo
  + local f
  + for f in "$@"
  + case "$f" in
  + '[' -x /docker-entrypoint-initdb.d/001-init-user-db.sh ']'
  + echo '/usr/local/bin/docker-entrypoint.sh: sourcing /docker-entrypoint-initdb.d/001-init-user-db.sh'
  + . /docker-entrypoint-initdb.d/001-init-user-db.sh
  /usr/local/bin/docker-entrypoint.sh: sourcing /docker-entrypoint-initdb.d/001-init-user-db.sh
  ++ psql -v ON_ERROR_STOP=1 --username postgres --dbname igix
  CREATE ROLE
  CREATE ROLE
  + echo
  + docker_temp_server_stop
  
  + PGUSER=postgres
  + pg_ctl -D /var/lib/postgresql/data -m fast -w stop
  waiting for server to shut down...2023-09-15 07:29:29.196 UTC [48] LOG:  received fast shutdown request
  .2023-09-15 07:29:29.199 UTC [48] LOG:  aborting any active transactions
  2023-09-15 07:29:29.200 UTC [48] LOG:  worker process: logical replication launcher (PID 55) exited with exit code 1
  2023-09-15 07:29:29.200 UTC [50] LOG:  shutting down
  2023-09-15 07:29:29.275 UTC [48] LOG:  database system is shut down
   done
  server stopped
  + unset PGPASSWORD
  + echo
  + echo 'PostgreSQL init process complete; ready for start up.'
  + echo
  
  PostgreSQL init process complete; ready for start up.
  
  + exec postgres -c max_connections=5000 -c shared_buffers=8GB
  2023-09-15 07:29:29.305 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
  2023-09-15 07:29:29.305 UTC [1] LOG:  listening on IPv6 address "::", port 5432
  2023-09-15 07:29:29.309 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
  2023-09-15 07:29:29.482 UTC [85] LOG:  database system was shut down at 2023-09-15 07:29:29 UTC
  2023-09-15 07:29:29.492 UTC [1] LOG:  database system is ready to accept connections
  
  
  ```

  