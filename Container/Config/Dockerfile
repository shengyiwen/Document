#define source mirror
FROM minor_jdk:1.8_231

#define the location of application
ENV APP_HOME "/yxg/webservice/evision-zuul"

#set workdir
WORKDIR $APP_HOME

#make directory
RUN bash -c "mkdir -p ./logs"

#volume current path
VOLUME $APP_HOME

#copy evision-zuul.jar to workspace
COPY target/evision-zuul-starter.jar $APP_HOME/evision-zuul.jar

#expose port
EXPOSE 7004

#exec java command and start the job
#prefer use jvm container config rather than xms and xmx
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-XX:MaxRAMFraction=1", "-XshowSettings:vm", \
  "-XX:NewSize=64m", "-XX:MetaspaceSize=128m", "-XX:MaxMetaspaceSize=128m", \
  "-XX:+UseConcMarkSweepGC", "-XX:CMSInitiatingOccupancyFraction=78", "-XX:+CMSClassUnloadingEnabled", \
  "-XX:ThreadStackSize=256", "-XX:+PrintGC", "-Xloggc:./logs/gc.log", \
  "-XX:+HeapDumpOnOutOfMemoryError", "-XX:HeapDumpPath=./logs/heap_dump.hprof", \
  "-jar", "evision-zuul.jar", ">/dev/null &"]