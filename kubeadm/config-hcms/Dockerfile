# Tomcat 9 + JDK 8 (Servlet 4.0 / Java EE 8). Jakarta EE 9+ WAR는 tomcat:10.1-jre17 등으로 변경
FROM tomcat:9-jdk8

# 외부 디렉토리를 ROOT로 사용하므로 WAR 파일 복사 불필요
# 호스트의 /home/jnet/tomcat-war-deploy/h-cms/webapp 디렉토리가 
# 컨테이너의 /usr/local/tomcat/webapps/external-root로 마운트됨
# WAR 배치: 루트 컨텍스트(/)로 서비스하려면 ROOT.war로 복사 (외부 디렉토리 미사용 시)
# COPY jnet.humanframe.web-2.2.1.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

