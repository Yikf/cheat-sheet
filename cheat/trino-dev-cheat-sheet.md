
This cheat fork from (Trino-dev-cheat-sheet)[https://gist.github.com/findepi/04c96f0f60dcc95329f569bb0c44a0cd], and thanks a lot @(findepi)[https://github.com/findepi]

#### quick build
```
./mvnw -pl '!:trino-server-rpm,!:trino-docs,!:trino-proxy,!:trino-verifier,!:trino-benchto-benchmarks' clean install -TC2 -nsu -DskipTests -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -Dair.check.skip-all=true -Dskip.npm -Dskip.yarn
```

#### run Trino in a container, quickly
```
docker rm -f trino; docker run --rm -it --name trino -p 8080:8080 trinodb/trino:361
```

#### run static code analysis
```
# checkstyle only
./mvnw -TC2 checkstyle:check@checkstyle

# all verifications, including Eerror Prone compiler
./mvnw clean verify -DskipTests -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -Dskip.npm -Dskip.yarn -nsu -P errorprone-compiler -pl '!:trino-server,!:trino-server-rpm,!:trino-docs'
```


#### quick build parser alone
```
./mvnw clean install -pl :trino-parser -TC2 -DskipTests -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -Dair.check.skip-all=true
```


#### kill and remove all docker containers
```
docker ps -a | tail -n+2 | grep . && docker ps -aq | xargs docker rm -f ; docker volume prune -f
```

#### docs
```
docs/build
```



#### extract exception stacktrace from query JSON
```
# Just top-level exception
jq -r ' .failureInfo | [ .type + ": " + .message, (.stack[] | "    at " + .) ][] ' < query.json

# Full stacktrace
python -c '
import sys
import json
def print_exc(exc, enclosing_stacktrace=(), caption="", prefix=""):
    print("%s%s%s: %s" % (prefix, caption, exc["type"], exc["message"]))
    if "errorCode" in exc:
        print("%s\t\tcode: %s, name: %s" % (prefix, exc["errorCode"]["code"],
            exc["errorCode"]["name"]))
    stack_trace = exc["stack"]
    our_stack = unique_stacktrace(stack_trace, enclosing_stacktrace)
    for l in our_stack:
        print("%s\tat %s" % (prefix, l))
    if len(our_stack) < len(stack_trace):
        print("%s\t..." % (prefix, ))
    for s in exc["suppressed"]:
        print_exc(s, stack_trace, "Suppressed: ", prefix + "\t")
    if "cause" in exc:
        print_exc(exc["cause"], stack_trace, "Caused by: ", prefix)
def unique_stacktrace(stack, enclosing):
    stack = list(stack)
    enclosing = list(enclosing)
    while stack and enclosing and stack[-1] == enclosing[-1]:
        stack.pop()
        enclosing.pop()
    return stack
qj = json.load(sys.stdin)
print_exc(qj["failureInfo"])
' \
 < query.json
```


#### restore Product Test Launcher (ptl) removed by a very important person
```
# this is required for commands show below
git restore --source=$(git log -1 --format=tformat:%H -- bin/ptl)^ -- bin/ptl
```


#### product test env just as product tests do
```
bin/ptl env up --environment singlenode
```

#### postgresql
```
bin/ptl env up --environment singlenode-postgresql --without-trino
# docker exec -itu postgres ptl-postgresql psql -U test -d test
```

#### sqlserver
```
bin/ptl env up --environment singlenode-sqlserver --without-trino
# docker exec -it ptl-sqlserver /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P SQLServerPass1
```

#### mysql
```
bin/ptl env up --environment singlenode-mysql --without-trino
# docker exec -it ptl-mysql mysql -u test -ptest -D test
```

#### `git bisect` with plain test
```
git bisect run bash -xeuc '
  ./mvnw clean install -am -pl :trino-tests -TC2 -DskipTests \
    -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -Dair.check.skip-all=true || exit 125
  ./mvnw test -pl :trino-tests -Dair.check.skip-all=true -Dtest=TestLocalQueriesA#test
'
```

#### `git bisect` with product tests
```
git bisect run bash -xeuc '
  { ./mvnw clean install -am -pl \!:trino-server-rpm,\!:trino-docs -DskipTests \
    -Dair.check.skip-all=true -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -TC2 || exit 125; } \
  && time bin/ptl test run --environment singlenode -- -g tpcds -x quarantine -t q98'
```

#### run product test quick
```
# TODO update this
#   java -jar presto-product-tests/target/presto-product-tests-*-executable.jar -c presto-product-tests/src/main/resources/sql-tests -g tpch -t q17
```

#### bring up product test env
```
# restore Product Test Launcher (ptl) removed by a very important person
git restore --source=$(git log -1 --format=tformat:%H -- bin/ptl)^ -- bin/ptl

# get a list of envs
bin/ptl env list

# start an env of your choice, exposing JVM debug port for Trino
bin/ptl env up --environment singlenode-hdp3 --debug

# run Trino CLI against the env
client/trino-cli/target/trino-cli-*-executable.jar --debug --server localhost:8080

# run Hive beeline 
docker exec -itu hive ptl-hadoop-master bash -l
echo "press Ctrl+R for bash history search and type 'beeline'"

# run Spark SQL (when dealing with an environment containing Spark image)
docker exec -it ptl-spark spark-sql
```

#### check all commits compile
```
git rebase master -x './mvnw clean package -pl \!:trino-server-rpm,\!:trino-docs -TC2 -DskipTests -Dmaven.javadoc.skip=true -Dmaven.source.skip=true'
```

#### sort commits for GitHub
```
git rebase master -x 'git commit --amend -C HEAD --date="$(date -R)" && sleep 1.05'
```

### rebase commits within PR

This makes GitHub's diff for force-push useful.

```
git rebase -i $(git merge-base head master)
```

#### network grepping
```
sudo ngrep -s0 -W byline -d lo0
```

