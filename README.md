--- PHoss.orig/list.c	Thu Jun  1 15:49:29 2000
+++ PHoss/list.c	Fri Sep 14 08:46:28 2001
@@ -1,5 +1,6 @@
 /* $Id: list.c,v 1.4 2000/06/01 13:49:20 fx Exp fx $ */
 #include <stdio.h>
+#include <stdlib.h>
 #include <string.h>
 #include <netinet/in.h>                 // for IPPROTO_bla consts
 #include <sys/socket.h>                 // for inet_ntoa()
--- PHoss.orig/main.c	Tue Jun 20 16:21:54 2000
+++ PHoss/main.c	Fri Sep 14 09:54:42 2001
@@ -1,6 +1,6 @@
 /* PHoss (PHenoelit's own security sniffer */
 /* $Id: main.c,v 1.13 2000/06/20 14:21:38 fx Exp fx $ */
-
+#include <ctype.h> 
 #include <stdio.h>
 #include <stdlib.h>
 #include <unistd.h>
@@ -90,6 +90,7 @@
 char *tcp_data;
 unsigned int tcp_data_length;
 
+char pcap_filename[255]="\0";
 
 /* functions */
 void print_found(SupportedProtos prot, char *cleardata);
@@ -118,33 +119,46 @@
 }
 
 int init_capture(char *device,unsigned int snaplength) {
-
-    if (device==NULL) {
-	VERBOSE(1,"device not set. Looking up ...\n");
-	if ((device=pcap_lookupdev(pcap_err))==NULL) {
-	    fprintf(stderr,"init_capture(): %s\n",pcap_err);
+    
+     if ( pcap_filename[0] ) {
+       VERBOSE(1,"Opening file ...\n");
+       if ( (cap = pcap_open_offline(pcap_filename ,pcap_err))==NULL  ) {
+           fprintf(stderr,"init_capture(): %s\n",pcap_err);
 	    return (-1);
-	}
-    }
-    if (verbose) printf("Device: %s\n",device);
-
-    VERBOSE(2,"Looking up network ...\n");
-    if (pcap_lookupnet(device,&network,&netmask,pcap_err)!=0) {
-	fprintf(stderr,"init_capture(): %s\n",pcap_err);
-	return (-1);
-    }
-
-    /* 1 = promiscuous mode , 0 = timeout */
-    VERBOSE(2,"opening network ...\n");
-    if ((cap=pcap_open_live(device,snaplength,1,0,pcap_err))==NULL) {
-	fprintf(stderr,"init_capture(): %s\n",pcap_err);
-	return (-1);
-    }
-
-    VERBOSE(2,"compiling filter ...\n");
-    if (pcap_compile(cap,&cfilter,filter,0/*no optim*/,netmask)!=0) {
-	capterror(cap,"compiler");
+       }
+       VERBOSE(2,"compiling filter ...\n");
+       if (pcap_compile(cap,&cfilter,filter,1,0)!=0) {
+	   capterror(cap,"compiler");
+       }
+    } else {
+       if (device==NULL) {
+	   VERBOSE(1,"device not set. Looking up ...\n");
+	   if ((device=pcap_lookupdev(pcap_err))==NULL) {
+	       fprintf(stderr,"init_capture(): %s\n",pcap_err);
+	       return (-1);
+	   }
+       }
+       if (verbose) printf("Device: %s\n",device);
+
+       VERBOSE(2,"Looking up network ...\n");
+       if (pcap_lookupnet(device,&network,&netmask,pcap_err)!=0) {
+	   fprintf(stderr,"init_capture(): %s\n",pcap_err);
+	   return (-1);
+       }
+
+       /* 1 = promiscuous mode , 0 = timeout */
+       VERBOSE(2,"opening network ...\n");
+       if ((cap=pcap_open_live(device,snaplength,1,0,pcap_err))==NULL) {
+	   fprintf(stderr,"init_capture(): %s\n",pcap_err);
+	   return (-1);
+       }
+    
+       VERBOSE(2,"compiling filter ...\n");
+       if (pcap_compile(cap,&cfilter,filter,0/*no optim*/,netmask)!=0) {
+	   capterror(cap,"compiler");
+       }
     }
+ 
 
     VERBOSE(2,"setting filter ...\n");
     if (pcap_setfilter(cap,&cfilter)!=0) {
@@ -249,7 +263,7 @@
     
     if (verbose>=3) {
 	memcpy(&MessageID,tcpd,4);
-	printf("LDAP Message ID: %d\n",MessageID);
+	printf("LDAP Message ID: %ld\n",MessageID);
 	printf("Choice: %x\n",(int)choice);
 	printf("LDAP Protocol Version: %d\n",tcpd[9]);
 	printf("Length of DN: %d\n",l);
@@ -814,8 +828,8 @@
 	printf("%s:%d\n",
 		inet_ntoa((struct in_addr)ip->daddr),
 		ntohs((unsigned short int)tcp->dest_port));
-	printf("\tSeq: %u\n",(unsigned long int)ntohl(tcp->seq_num));
-	printf("\tAck: %u\n",(unsigned long int)ntohl(tcp->ack_num));
+	printf("\tSeq: %lu\n",(unsigned long int)ntohl(tcp->seq_num));
+	printf("\tAck: %lu\n",(unsigned long int)ntohl(tcp->ack_num));
     }
     if (
 	    (phead->caplen-ETHLENGTH-
@@ -915,10 +929,12 @@
 
     u_char *packet,*pcap_packet;
     SupportedProtos identification;
-
+    cap_interrupted=0;
+    
     while (cap_interrupted==0) {
+        
 	if ((pcap_packet=(u_char *)pcap_next(cap,pcap_head))==NULL) continue;
-
+        
 	/* make sure it is our own data */
 	phead=smalloc(sizeof(struct pcap_pkthdr));
 	memcpy(phead,pcap_head,sizeof(struct pcap_pkthdr));
@@ -952,6 +968,7 @@
            "-l XX\tSet capture length to this value (default 1525)\n"
            "-i int\tUse this interface\n"
            "-f xx\tSet packet filter. See tcpdump(1) for more\n"
+           "-r filename\tread input from file\n"
            "-L \tmake output linebuffered\n");
     exit(0);
 }
@@ -969,7 +986,7 @@
 
     printf("%s\n%s\n",PROGRAM_NAME,PROGRAM_VERSION);
 
-    while ((option=getopt(argc,argv,"PLpvl:i:f:"))!=EOF) {
+    while ((option=getopt(argc,argv,"PLpvl:i:f:r:"))!=EOF) {
 	switch(option) {
 	    case 'v': verbose++;
 		      break;
@@ -996,6 +1013,8 @@
 	    case 'i': net_dev=(char *)smalloc(strlen(optarg)+1);
 		      strcpy(net_dev,optarg);
 		      break;
+            case 'r': strncpy(pcap_filename,optarg,255);
+	              break;
 	    case 'l': if ((slength=atoi(optarg))<24) {
 			  fprintf(stderr,"Useless capture length: %d - ignored\n",slength);
 			  slength=DEF_CAPLENGTH;
@@ -1003,7 +1022,7 @@
 	    default: usage();
 	}
     }
-
+    
     if (init_capture(net_dev,slength)==0) {
 	// rest of the program