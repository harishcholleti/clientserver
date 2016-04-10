#include <time.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#define MAXLINE 128000
#define LISTENQ 10
 void itoa(int number,char* buff)
{
	char buffer[12];
	snprintf(buffer, 12,"%d",number);
	strcpy(buff,buffer);
}


void read_command(int client_fd,char* cmd) {
	
	int n = read(client_fd,cmd,3);
        if(n > 0) {
		printf("Commmand received : %c%c%c\n",cmd[0],cmd[1],cmd[2]);
			
	}
}
int get_icommand(char* cmd) {
	if(strcmp("rls",cmd) == 0) return 0;
	if(strcmp("get",cmd) == 0) return 1;
	if(strcmp("put",cmd) == 0) return 2;
	if(strcmp("bye",cmd) == 0) return 3;
	
return -1;
}


void get_filenames(char** cmd_data_buff,int* buff_size) {

        FILE *pipein_fp1,*pipein_fp2;
        char readbuf[120]= {0};
	int cnt = 0;
	int sz=0;
	char *pp = NULL;
	
        /* Create one way pipe line with call to popen() */
        if (( pipein_fp1 = popen("ls -l", "r")) == NULL)
        {
                perror("popen");
                exit(1);
        }

        /* Processing loop */
	while(fgets(readbuf, 120, pipein_fp1))
                sz=sz+strlen(readbuf);
	
        /* Close the pipes */
        pclose(pipein_fp1);
	*cmd_data_buff = (char*) malloc(sz+10);
	pp = *cmd_data_buff;

        if (( pipein_fp2 = popen("ls -l", "r")) == NULL)
        {
                perror("popen");
                exit(1);
       	}

        /* Processing loop */
	while(fgets(readbuf, 120, pipein_fp2)) {
		strcpy(&pp[cnt],readbuf);			
		cnt+= strlen(readbuf);	
	}
	*buff_size = cnt;
        pclose(pipein_fp2);

}	

char* assemble_reply(char* cmd_data_buff,int buff_size,int *reply_size) {
//1234:data
	char num[8] = {0};
	char* final_buff = NULL;
	itoa(buff_size,num);

	*reply_size = strlen(num)+1+buff_size;

	final_buff = (char*)malloc(strlen(num)+1+buff_size);
	memcpy(final_buff,num,strlen(num));
	final_buff[strlen(num)] = ':';
	memcpy(&final_buff[strlen(num)+1],cmd_data_buff,buff_size);
	return final_buff;
}

void get_filename(int client_fd,char* file_name) {
	char chr = 0;
	int cnt=0;	
	read(client_fd,&chr,1);
	if(chr != ':') {
		printf("char read = %c\n",chr);
		return;
	}
	printf("char read = %c\n",chr);

	read(client_fd,&chr,1);
	file_name[cnt] = chr;
	while(chr != '\0') {
		int n = 0;	
		printf("%d\n",chr);		
		 n = read(client_fd,&chr,1);
		if(n <= 0) break;
		cnt++;
		file_name[cnt] = chr;
	}
	printf("file name = %s\n",file_name);	
}

char* get_file_data(char* file_name, int *len) {

	char* buff = NULL;	
	FILE* fp = fopen(file_name,"r");
	if(fp == NULL)
		return NULL;
	fseek(fp,0,SEEK_END);
	*len = ftell(fp);
	fseek(fp,0,SEEK_SET);
        buff = (char*)malloc(*len); 

	fread(buff,*len,1,fp);
return buff;
}	

void* handle_client(void *p) {

	int client_fd = *(int*)p;
	char cmd[12] = {0};
	int icmd = 0;
	char *cmd_data_buff = NULL;
	int buff_size = 0;
	char* reply = NULL;
	int reply_size = 0;
	char file_name[256] = {0};
	int file_len = 0;

	/* keep on spinning until I get command from user like rls,get, put or bye*/
	while(1) {
	while(strlen(cmd) == 0 ){ 	
		read_command(client_fd,cmd);
                if(strcmp(cmd,"bye") == 0) { 
			printf("Exiting client thread ...\n");
			return;
		}
	}
	
				
	icmd = get_icommand(cmd);
 	switch(icmd) {

		case 0: //rls
		printf("Processing command rls ... \n");
		get_filenames(&cmd_data_buff,&buff_size);
		reply = assemble_reply(cmd_data_buff,buff_size,&reply_size);
		write(client_fd,reply,reply_size);
		free(cmd_data_buff);
		free(reply);
		break;
		case 1: //get
		memset(file_name,0x00,256);
		get_filename(client_fd,file_name);
		if(strlen(file_name) == 0) { 
			printf("Bad command !\n");
			write(client_fd,"ERROR",5);
		}
		cmd_data_buff = get_file_data(file_name,&file_len);
		if(!cmd_data_buff) {
			printf("File does not exit !\n");
			write(client_fd,"ERROR",5);
		}	
		reply = assemble_reply(cmd_data_buff,file_len,&reply_size);
		write(client_fd,reply,reply_size);
		break;
		case 2: //put
		break;
		default:
		printf("Not a command !\n");
		write(client_fd,"ERROR",5);
		break;	
	}
	       memset(cmd,0x00,12);

	}			

}	

int main(int argc, char **argv)
{
	int addison, listenfd, connfd;
	pthread_t thread;
	struct sockaddr_in servaddr;
        int n=0;
	listenfd = socket(AF_INET, SOCK_STREAM, 0);
	memset((void*)&servaddr,0x00,sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(4300);

	bind(listenfd,(struct sockaddr *)&servaddr, sizeof(servaddr));

	listen(listenfd, LISTENQ);

	for ( ; ; ) {
		connfd = accept(listenfd, (struct sockaddr *) NULL, NULL);
		printf("Client connected ... \n"); 
      		pthread_create(&thread,NULL,handle_client,&connfd);
	} 
return 0;
}
