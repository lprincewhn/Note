近期在工作中遇到了这个问题，所以特地去研究了一下TCP会话的关闭机制和解决方法。TCP会话以及TIME_WAIT状态的机制如下图所示。简单而言，TCP_WAIT状态的存在的目的是：防止当“主动关闭端”的最后一个ACK包在网络中丢失而导致“被动关闭端”重发FIN包时，“主动关闭端”需要能正确处理这个重发的FIN包。






如上图所示，如果左侧的“Active close”发送了最后一个ACK(K+1)包之后没有TIME_WAIT状态，直接把TCP会话关闭，会有以下两个问题:
如果ACK(K+1)包丢失了，“Passive close”重新发送了FIN(K)包，由于此时“Active close”已经不存在该回话，无法再确认FIN(K)。
更严重的问题是，如果收到重发的FIN(K)包的时候，“Active close”如果已经使用相同的socket建立了一个新的会话，将导致重发FIN(K)包被发送到新的会话中处理，干扰了新的会话。
RFC793中关于TCP会话如何关闭，列举了以下两种场景：



      TCP A                                                TCP B

  1.  ESTABLISHED                                          ESTABLISHED

  2.  (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT

  3.  FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT

  4.                                                       (Close)
      TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK

  5.  TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --> CLOSED

  6.  (2 MSL)
      CLOSED                                                      

                         Normal Close Sequence

                               Figure 13.

  

      TCP A                                                TCP B

  1.  ESTABLISHED                                          ESTABLISHED

  2.  (Close)                                              (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  ... FIN-WAIT-1
                  <-- <SEQ=300><ACK=100><CTL=FIN,ACK>  <--
                  ... <SEQ=100><ACK=300><CTL=FIN,ACK>  -->

  3.  CLOSING     --> <SEQ=101><ACK=301><CTL=ACK>      ... CLOSING
                  <-- <SEQ=301><ACK=101><CTL=ACK>      <--
                  ... <SEQ=101><ACK=301><CTL=ACK>      -->

  4.  TIME-WAIT                                            TIME-WAIT
      (2 MSL)                                              (2 MSL)
      CLOSED                                               CLOSED

                      Simultaneous Close Sequence

                               Figure 14.



我分别使用NFS，FTP, HTTP三种协议进行了实验，

来源：  https://datatracker.ietf.org/doc/rfc793/?include_text=1