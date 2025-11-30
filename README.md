# 校赛技术文档

## 1. DOMjudge

### 1.1 服务器
首先现场赛需要~~自掏腰包~~一台服务器，用于部署 DOMjudge。2024 年新生赛与 2025 年校赛选用的是 ecs.hfc8i.2xlarge 的实例（大概 3 元/小时），8 核 16 G。另外注意选服务器的地域时，建议选中国香港（git 的时候需要翻墙）。

系统选择 Ubuntu 22.04。记得勾选分配公网 IPv4 地址，带宽计费方式选择按使用流量，带宽峰值选 100 Mbps。设置密码后下单即可。

记住公网 IP，这就是选手访问的比赛地址。

选择远程连接，workbench 远程连接即可。

### 1.2 DOMjudge 的部署

#### 1.2.1 系统准备

先执行：
```bash
sudo vim /etc/default/grub
```

然后把 `GRUB_CMDLINE_LINUX_DEFAULT=""` 改为：
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1 isolcpus=2 systemd.unified_cgroup_hierarchy=0"
```

再执行：
```bash
sudo update-grub
```

然后重启服务器：
```bash
sudo reboot
```
这一步可以直接在控制台重启。

#### 1.2.2 安装 docker 与 docker-compose

首先执行：
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

然后执行：
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

执行：
```bash
curl -SL https://github.com/docker/compose/releases/download/v2.40.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

接下来创建：
```bash
sudo mkdir -p /etc/docker-compose
```

创建：
```bash
sudo vim /etc/systemd/system/docker-compose@.service
```

输入以下内容：
```
[Unit]
Description=%i service deployed with docker compose
Requires=docker.service
After=docker.service

[Service]
user=root
Type=simple
WorkingDirectory=/etc/docker-compose/%i
ExecStart=/usr/local/bin/docker compose up --remove-orphans

[Install]
WantedBy=multi-user.target
```

执行：
```bash
sudo systemctl daemon-reload
```

#### 1.2.3 安装 DOMjudge

首先执行：
```bash
cd /etc/docker-compose/
sudo git clone https://github.com/Sy-SU/domjudge-compose.git domjudge
cd domjudge
```

**需要修改 `database.secret` 文件中 `<GET THIS FROM TERMINAL>`。建议使用两个强密码。**
```bash
sudo vim database.secret
```

然后运行数据库和后端系统，获取 judgehost 的密钥：
```bash
sudo docker compose up -d dj-mariadb domserver
```

获取 judgehost 的密钥：
```bash
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret
```

然后替换 `judgehost.secret` 文件中 `<GET THIS FROM TERMINAL>` 的内容。
```bash
sudo vim judgehost.secret
```

接下来修改 `docker-compose.yml` 中的：
```txt
...
  judgehost:
  ...
      deploy:
          ...
          replicas: 4 # CHANGE THIS
          ...
...
```

按照我校的评测情况，4 个 judgehost 是够的。

然后运行：
```bash
sudo docker compose up -d
```

接下来完成数据持久化：

```bash
# 在 dom 目录下执行

# 创建必要的目录
mkdir -p database backups domserver_data/{public/images,public/flags,var}


# 从运行的容器中复制默认数据到宿主机
echo "正在复制 DOMServer 默认数据到宿主机..."

# 复制图片和标志文件
docker cp domserver:/opt/domjudge/domserver/webapp/public/images/. domserver_data/public/images/
docker cp domserver:/opt/domjudge/domserver/webapp/public/flags/. domserver_data/public/flags/

# 复制变量数据
docker cp domserver:/opt/domjudge/domserver/webapp/var/. domserver_data/var/

# 验证数据复制成功
echo "验证复制的数据:"
ls -la domserver_data/public/images/
ls -la domserver_data/public/flags/

echo "数据复制完成！"
```

接下来修改 `docker-compose.yml`：

```bash
sudo vim docker-compose.yml
```

```yml
    # volumes:
    #   - ./backups:/opt/domjudge/domserver/backups
    #   - ./domserver_data/public/images:/opt/domjudge/domserver/webapp/public/images
    #   - ./domserver_data/public/flags:/opt/domjudge/domserver/webapp/public/flags
    #   - ./domserver_data/var:/opt/domjudge/domserver/webapp/var 
```

将上面这一部分取消注释即可。

通过运行：

```bash
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

获取管理员密码。管理员的账户为 `admin`。然后重启，因为现在数据挂载还没有生效：

```bash
docker-compose down
docker-compose up -d
```

最后执行：

```bash
sudo systemctl enable docker-compose@domjudge.service
```
设置开机启动。

通过公网 IP 即可访问。

#### 1.2.4 DOMjudge 的配置

1. 首先进入 `jury/config`，在 `External systems` 中把 `Data source` 切换成 `configuration data external`。然后点击 `Save changes` 保存更改。
2. 在 `Scoring` 中：`Results remap` 中将 `no-output` 与 `output-limit` 映射为 `wrong-answer`，`memory-limit` 映射成 `run-error`。然后点击 `Save changes` 保存更改。
3. 在 `Judging` 中：`Sorucefiles limit` 改为 `1`，`Timelimit overshoot` 改为 `15s|500%`，`Output limit` 改为 `81920`
4. 在 `Clarifications` 中加一下常用的 response，比如：“请仔细读题”，“不予回答”等。
5. 在 `Display` 中：`Show flags` 关掉，`Show affiliation logos` 开启，`Allow team submission download` 开启，`Show teams on scoreboard` 改为 `After login`。
6. 修改 `jury/executables/c` 的 run 为：
```txt
#!/bin/sh

# C compile wrapper-script for 'compile.sh'.
# See that script for syntax and more info.

DEST="$1" ; shift
MEMLIMIT="$1" ; shift

# Add -DONLINE_JUDGE or -DDOMJUDGE below if you want it make easier for teams
# to do local debugging.

# -x c:     Explicitly set compile language to C (no C++ nor object files
#           autodetected by extension)
# -Wall:    Report all warnings
# -O2:      Level 2 optimizations (default for speed)
# -static:  Static link with all libraries
# -pipe:    Use pipes for communication between stages of compilation
# -lm:      Link with math-library (has to be last argument!)
gcc -x c -DONLINE_JUDGE -std=c11 -Wall -O2 -static -pipe -o "$DEST" "$@" -lm
exit $?
```
7. 修改 `jury/executables/cpp` 的 run 为：
```txt
#!/bin/sh

# C++ compile wrapper-script for 'compile.sh'.
# See that script for syntax and more info.

DEST="$1" ; shift
MEMLIMIT="$1" ; shift

# Add -DONLINE_JUDGE or -DDOMJUDGE below if you want it make easier for teams
# to do local debugging.

# -x c++:   Explicitly set compile language to C++ (no object files or
#           other languages autodetected by extension)
# -Wall:    Report all warnings
# -O2:      Level 2 optimizations (default for speed)
# -static:  Static link with all libraries
# -pipe:    Use pipes for communication between stages of compilation
g++ -x c++ -DONLINE_JUDGE -std=c++20 -Wall -O2 -static -pipe -o "$DEST" "$@"
exit $?
```
8. 修改 `jury/executables/java_javac_detect` 的 run 为：
```txt
#!/bin/sh

# Java compile wrapper-script for 'compile.sh'.
# See that script for syntax and more info.
#
# This script byte-compiles with the javac compiler and
# generates a shell script to run it with the java interpreter later.
# It autodetects the main class name and optionally renames the source
# file if the class is public.
#
# This script requires that java is installed in the chroot.

DEST="$1" ; shift
MEMLIMIT="$1" ; shift
MAINSOURCE="$1"
MAINCLASS=""
COMPILESCRIPTDIR="$(dirname "$0")"

# Stack size in the JVM in KB. Note that this will be deducted from
# the total memory made available for the heap.
MEMSTACK=65536

# Amount of memory reserved for the Java virtual machine in KB. The
# default below is just above the maximum memory usage of current
# versions of the jvm, but might need increasing in some cases.
MEMJVM=65536

MEMRESERVED=$((MEMSTACK + MEMJVM))

# Calculate Java program memlimit as MEMLIMIT - max. JVM memory usage:
MEMLIMITJAVA=$((MEMLIMIT - MEMRESERVED))

if [ $MEMLIMITJAVA -le 0 ]; then
	echo "internal-error: total memory $MEMLIMIT KiB <= $MEMJVM + $MEMSTACK = $MEMRESERVED KiB reserved for JVM and stack leaves none for heap."
	exit 1
fi

# Java needs filename to match main class:
if [ -z "$ENTRY_POINT" ]; then
	[ -n "$DEBUG" ] && echo "Debug: no ENTRY_POINT provided, trying to detect main class."
else
	[ -n "$DEBUG" ] && echo "Debug: using main class provided by ENTRY_POINT='$ENTRY_POINT'."
	MAINCLASS="$ENTRY_POINT"
fi

TMPFILE=$(mktemp domjudge_javac_output.XXXXXX) || exit 1

# Byte-compile:
javac -encoding UTF-8 -sourcepath . -d . "$@" 2> "$TMPFILE"
EXITCODE=$?

cat $TMPFILE
rm -f $TMPFILE

[ "$EXITCODE" -ne 0 ] && exit $EXITCODE

if [ -z "$MAINCLASS" ]; then
	# Look for class that has the 'main' function:
	CLASSNAMES="$(find ./* -type f -regex '^.*\.class$' \
	            | sed -e 's/\.class$//' -e 's/^\.\///' -e 's/\//./g')"
	MAINCLASS=$(java -cp "$COMPILESCRIPTDIR" DetectMain "$(pwd)" $CLASSNAMES)
	EXITCODE=$?

	# Report the entry point, so it can be saved, e.g. for later replay:
	echo "Info: detected entry_point: $MAINCLASS"

	[ "$EXITCODE" -ne 0 ] && exit $EXITCODE
else
	# Check if entry point is valid
	java -cp "$COMPILESCRIPTDIR" DetectMain "$(pwd)" $MAINCLASS > /dev/null
	EXITCODE=$?
	[ "$EXITCODE" -ne 0 ] && exit $EXITCODE
fi

# Write executing script:
# Executes java byte-code interpreter with following options
# -Xmx: maximum size of memory allocation pool
# -Xms: initial size of memory, improves runtime stability
# -XX:+UseSerialGC: Serialized garbage collector improves runtime stability
# -Xss${MEMSTACK}k: stack size as configured above
# -Dfile.encoding=UTF-8: set file encoding to UTF-8
cat > "$DEST" <<EOF
#!/bin/sh
# Generated shell-script to execute java interpreter on source.

# Detect dirname and change dir to prevent class not found errors.
if [ "\${0%/*}" != "\$0" ]; then
	cd "\${0%/*}"
fi

# Add -DONLINE_JUDGE or -DDOMJUDGE below if you want it make easier for teams
# to do local debugging.

exec java -DONLINE_JUDGE -Dfile.encoding=UTF-8 -XX:+UseSerialGC -Xss${MEMSTACK}k -Xms${MEMLIMITJAVA}k -Xmx${MEMLIMITJAVA}k '$MAINCLASS' "\$@"
EOF

chmod a+x "$DEST"

exit 0
```
9. 修改 `jury/executables/py3` 的 run 为：
```txt
#!/bin/sh

# Python3 compile wrapper-script for 'compile.sh'.
# See that script for syntax and more info.
#
# This script does not actually "compile" the source, but writes a
# shell script that will function as the executable: when called, it
# will execute the source with the correct interpreter syntax, thus
# allowing this interpreted source to be used transparently as if it
# was compiled to a standalone binary.
# Note that from version 8.0.0 the default Python 3 interpreter is
# pypy 3 instead of CPython 3.
#
# This script requires that pypy3 is installed in the chroot.

DEST="$1" ; shift
MEMLIMIT="$1" ; shift
MAINSOURCE="${ENTRY_POINT:-$1}"

# Report the entry point, so it can be saved, e.g. for later replay:
if [ -z "$ENTRY_POINT" ]; then
    echo "Info: detected entry_point: $MAINSOURCE"
fi

# Check syntax
pypy3 -m py_compile "$@"
EXITCODE=$?
[ "$EXITCODE" -ne 0 ] && exit $EXITCODE
rm -f -- *.pyc
rm -rf -- __pycache__

# Check if entry point is valid
if [ ! -r "$MAINSOURCE" ]; then
    echo "Error: main source file '$MAINSOURCE' is not readable" >&2
    exit 1
fi

# Write executing script:
cat > "$DEST" <<EOF
#!/bin/sh
# Generated shell-script to execute python interpreter on source.

# Detect dirname and change dir to prevent class not found errors.
if [ "\${0%/*}" != "\$0" ]; then
	cd "\${0%/*}"
fi

# set non-existing HOME variable to make python happy
export HOME=/does/not/exist

# Uncomment the line below if you want it make easier for teams to do local
# debugging.
export ONLINE_JUDGE=1 DOMJUDGE=1

exec pypy3 "$MAINSOURCE" "\$@"
EOF

chmod a+x "$DEST"

exit 0
```
10. 在 `jury/categories/4/edit` 中把 obeservers 的 Sortorder 设置成 0。

#### 1.2.5 导入题目

1. 把 user:admin 的 Team 设置为 DOMjudge，Roles 勾选 Administrative User, Jury User, Team Member, Balloon runner
2. 在 Polygon 中，对每个题 package，然后 download Linux 版本的 package。
3. 执行 `pipx install p2d` 下载 `p2d` 工具。
4. 将题目重命名为 `A.zip` 的格式。执行：
```bash
p2d --code A --color "#FF0000" -o DJ_A.zip A.zip
p2d --code B --color "#FF0000" -o DJ_B.zip B.zip
p2d --code C --color "#FF0000" -o DJ_C.zip C.zip
p2d --code D --color "#FF0000" -o DJ_D.zip D.zip
p2d --code E --color "#FF0000" -o DJ_E.zip E.zip
p2d --code F --color "#FF0000" -o DJ_F.zip F.zip
p2d --code G --color "#FF0000" -o DJ_G.zip G.zip
p2d --code H --color "#FF0000" -o DJ_H.zip H.zip
p2d --code I --color "#FF0000" -o DJ_I.zip I.zip
p2d --code J --color "#FF0000" -o DJ_J.zip J.zip
p2d --code K --color "#FF0000" -o DJ_K.zip K.zip
p2d --code L --color "#FF0000" -o DJ_L.zip L.zip
p2d --code M --color "#FF0000" -o DJ_M.zip M.zip
```
5. 新建一个 contest，external ID 设成 contest，short name 设置成 contest，Name 设置成比赛的名称：华中农业大学第十五届程序设计竞赛（新生赛）。Activate time 设置成当场比赛开始前两个小时（选手可以进场的时间，设早一点也可以），Start time 设置成正式赛的时间，Scoreboard freeze time 设置成 +04:00:00，End time 设置成 +05:00:00，Scoreboard unfreeze time 设置成 +08:00:00。Medals enabled 设置成 Yes，Medal categories 设置成 Participants，Gold/Silver/Bronze medals 分别设置成 5/10/15，Open contest to all teams 设置成 No，Team categories 勾选 System, Participants, Obeservers，在 Problemset document 中上传题面。然后就可以把 demo contest 删掉了。

6. 在 `jury/import-export#problemarchive` 中，选择 DJ_X.zip，上传即可，然后手动修改气球颜色。这个时候 package 内的提交会自动测，记得看一下时限需不需要调整。

#### 1.2.6 导入队伍
1. 首先下载报名表的收集结果，只需要 姓名、学校、电子邮箱、年级。记得手动去一下重，然后把年级标为对应的选手类别（正式/打星）。按照选手类别为第一关键字，学校为第二关键字，姓名为第三关键字排序。
2. ![](image.png)把选手信息表添加上述信息（teamid 是选手登录的账号，location 是选手的位置，password 是选手的密码）。执行以下脚本：
```py
import json
from collections import OrderedDict
import pandas as pd  # pip install pandas openpyxl

# ===== 配置 =====
XLSX_FILE = "Test.xlsx"
RAW_JSON_FILE = "input.json"
TEAMS_FILE = "teams.json"
ORG_MAP_FILE = "org_map.json"
ORGANIZATIONS_FILE = "organizations.json"
ACCOUNTS_FILE = "accounts.json"

# 选手类别 -> group_id
GROUP_MAP = {
    "Participants": "participants",
    "Observers": "observers",
}


def read_xlsx_to_records(xlsx_path: str):
    """读取 xlsx -> 记录列表（每行一个 dict）"""
    df = pd.read_excel(xlsx_path, dtype=str)
    df = df.fillna("")
    return df.to_dict(orient="records")


def build_org_map(records):
    """学校 -> organization_id (SCHOOL-x)"""
    org_map = OrderedDict()
    next_id = 1
    for rec in records:
        school = rec.get("学校")
        if school and school not in org_map:
            org_map[school] = f"SCHOOL-{next_id}"
            next_id += 1
    return org_map


def build_organizations(org_map):
    """
    根据 org_map 生成 organizations.json 元素列表：
    id: INST-x
    name: 学校名
    formal_name: 学校名
    country: "CHN"
    """
    orgs = []
    for school, inst_id in org_map.items():
        orgs.append({
            "id": inst_id,
            "name": school,
            "formal_name": school,
            "country": "CHN",
        })
    return orgs


def build_teams(records, org_map):
    """从原始记录 + 学校映射构造 teams.json"""
    teams = []
    for rec in records:
        school = rec.get("学校")
        org_id = org_map.get(school)
        group_id = GROUP_MAP.get(rec.get("选手类别"), "0")

        team = {
            "id": rec.get("teamid"),
            "group_ids": [group_id],
            "name": rec.get("姓名"),
            "display_name": rec.get("姓名"),
            "organization_id": org_id,
            "location": {
                "description": rec.get("location"),
            },
        }
        teams.append(team)
    return teams


def build_accounts(records):
    """
    生成 accounts.json：
    - id: teamid
    - username: teamid
    - password: password
    - type: "team"
    - team_id: teamid
    """
    accounts = []
    for rec in records:
        team_id = rec.get("teamid")
        pwd = rec.get("password")
        acc = {
            "id": team_id,
            "username": team_id,
            "password": pwd,
            "type": "team",
            "team_id": team_id,
        }
        accounts.append(acc)
    return accounts


def main():
    # 1. XLSX -> records
    records = read_xlsx_to_records(XLSX_FILE)

    # 2. 原始中间 json
    with open(RAW_JSON_FILE, "w", encoding="utf-8") as f:
        json.dump(records, f, ensure_ascii=False, indent=2)

    # 3. 学校 -> INST-x 映射
    org_map = build_org_map(records)
    with open(ORG_MAP_FILE, "w", encoding="utf-8") as f:
        json.dump(org_map, f, ensure_ascii=False, indent=2)

    # 4. organizations.json
    orgs = build_organizations(org_map)
    with open(ORGANIZATIONS_FILE, "w", encoding="utf-8") as f:
        json.dump(orgs, f, ensure_ascii=False, indent=2)

    # 5. teams.json
    teams = build_teams(records, org_map)
    with open(TEAMS_FILE, "w", encoding="utf-8") as f:
        json.dump(teams, f, ensure_ascii=False, indent=2)

    # 6. accounts.json
    accounts = build_accounts(records)
    with open(ACCOUNTS_FILE, "w", encoding="utf-8") as f:
        json.dump(accounts, f, ensure_ascii=False, indent=2)

    print("生成完成：")
    print(f" - {RAW_JSON_FILE}")
    print(f" - {ORG_MAP_FILE}")
    print(f" - {ORGANIZATIONS_FILE}")
    print(f" - {TEAMS_FILE}")
    print(f" - {ACCOUNTS_FILE}")


if __name__ == "__main__":
    main()
```
在 `jury/import-export` 中的 `Teams & groups` 一栏，通过 `Import JSON / YAML` 的方式依次选择 `organizations.json`、`teams.json`、`accounts.json`，即可完成队伍导入。

### 1.3 滚榜
滚榜采用 icpc-tools 中的 resolver 工具。推荐使用 2.6.1160 版本。

首先 finalize 比赛，访问比赛的 api，导出 event-feed：
```txt
http://<domjudge url>/api/contests/<cid>/event-feed?stream=false
例如: http://47.243.212.3/api/contests/contest/event-feed?stream=false
```

将 event-feed 后缀改为 `.json`。手动去除所有 data 为 null 的字段。然后将工作目录切换到 `resolver/` 下。执行：
```bash
./award.sh
```

通过以下脚本导出需要的图片，在 `resolver/` 下新建 `cdp/`，把 `organizationsd/` 和 `teams/` 放进去。

选择 Disk，然后选择 `event-feed.json`。首先将 templates 中替换为：
```txt
{"id":"gold-medal","parameters":{"numTeams":"5"}}
{"id":"silver-medal","parameters":{"numTeams":"10"}}
{"id":"bronze-medal","parameters":{"numTeams":"15"}}
{"id":"first-to-solve-*"}
{"id":"honors-mention","parameters":{"percentileTop":"50","percentileBottom":"100"}}
```

执行：
```bash
ICPC_FONT="Microsoft Yahei" ./resolver.sh cdp --display_name "{org.formal_name} --{team.display_name}" --singleStep 70
```
即可。

