#New Root Major Freeze
##�°�major freeze��ɰ�major freeze�ĶԱ�
###�ɰ�major freeze���
�ھɰ��ʵ���У�major freeze���⻧Ϊ��λ��������⻧����ִ��major freeze�����ÿһ���⻧������partition���ɰ�major freezeͨ�����׶��ύ�ķ�ʽʵ�֣�����RSΪscheduler��paritcipantsΪ��ǰ�⻧������partition������RSû�е�����д��־�ӿڣ�ʵ����ѡȡpartition key��С��partition��Ϊcoordinator����ͨ��coordinator partition����־��ʶ���׶��ύ�Ľ��ȡ�ִ��major freeze�Ĺ��������е�participant partitions��Ҫͣ�������major freeze������ͣ�����ʱ����participant partitions����������أ���participant partitions������������ʱ���Ἣ��Ӱ��ҵ��
###�°�major freeze
Ϊ�˽���major freeze��ҵ���Ӱ�죬����major freeze��ִ��ʱ�䣬��major freeze��ʵ�ֽ������޸ģ��°�major freeze��Ȼ�������׶��ύʵ�֣�����RS��Ϊ���׶��ύ��coordinator��paritcipantsΪ��Ⱥ�е�����observer���������ʵ��ϸ���п��ܿ�����participantҲ�����Ǽ�Ⱥ�еĲ��ֲ����ߣ�������coordinator(RS)ͨ��д__all_core_table�����ʽ��¼���׶��ύ��־��participants��observer����д��־��participants�����׶��ύ״ֻ̬��¼���ڴ��С��°�major freeze��ִ�й�����ͬ����Ҫͣ����ֻ��������⣩��ͣ��������observer�յ�RS��prepare freeze����󣬲���observer�յ�������commit freeze��abort freeze�������ͣ������observer��Ϊmajor freeze���׶��ύ��participants�����Դ�󽵵�major freeze������ͣ�����ʱ�䣬��������major freeze��ҵ���Ӱ�졣
###����major freeze�Ĳ�ͬ
* �ɰ�major freeze��partition��Ϊ���׶��ύ��participants��major freezeִ�гɹ������е�partition���Ѿ������µ�memstore��ֱ�����������partition�������ݰ汾��
* �°�major freeze��observer��Ϊ���׶��ύ��participants��major freezeִ�гɹ���ֻ��observer����ά���Ķ�����Ϣ�����˸ı䣬observer�ϵ�partition��û�������µ�memstore��partition�������ݰ汾����Ҫ�������ƴ�������ˣ��°汾��major freezeʵ���ϱ�����Ϊ�˶���observer��partition���汾�������̣�partition���汾�ķ���Ϊ��leader partition��֪������active memtable�İ汾��observer�Ķ���汾�Ĳ��죬�����°汾memtable��д�������ݰ汾����־��follower partitionͨ���ط���leaderд�µ����汾��־��ɡ�

##�°�Root major freeze��ʵ��
###major freeze��״̬ת��
ͨ�����µ�һ����Ԫ������ʾĳ�����ݰ汾��major freeze��״̬�����ǳ�֮Ϊ
frozen_status=��frozen_version,frozen_timestamp,freeze_status,schema_version��
����frozen_version��ʾ����汾�ţ�frozen_timestamp��ʾ���major freeze�Ķ���ʱ�����freeze_status��ʾ���ζ�������׶��ύ�Ľ��ȣ���ѡȡֵ��INIT,PREPARED_SUCCEED��COMMIT_SUCCEED��schema_version��ʾ���ζ����schema_version��
����frozen_status=��5,10000,COMMIT_SUCCEED,10000����ʾ���ݰ汾Ϊ5��major freeze�Ѿ��ύ�ɹ������ҵ�ʱ��ʱ���Ϊ10000��schema_versionΪ10000����ͬ�汾��frozen_status֮���״̬ת��ͼ������ʾ������ʡ��frozen_timestamp��schema_version��V��ʾ�汾�ţ���
```nohighlight

+----------+      prepare       
| V,COMMIT |------------>---------------+
+----------+                            |
                                        |
                                        | 
+----------+      prepare       +--------------+      commit          +--------------+
| V+1,INIT |------------------->| V+1,PREPARED |--------------------->| V+1,COMMITED |
+----------+                    +--------------+                      +--------------+
  ^                                     |
  |           abort                     |
  +---------------------<---------------+
```
���°��major freeze�У�ÿһ��observer��ά��
###major freeze�������ڲ���
* \_\_all\_core\_table��  
�������ᵽ��major freeze������coordinator��RS����Ҫд��־��¼���׶��ύ�Ľ��ȣ�����RSû��ר�ŵ�дclog��־�ӿڣ�RS��д��־��ʽΪ��¼\_\_all\_core\_table�ڲ���RS��������observer����prepare requestǰдprepare record���ڷ���commit requestǰдcommit record���ڷ���abort recordǰЩabort record��observer���յ�prepare request���ͣ����ʹ����ͨsql���޷�����\_\_all\_core\_table�������ṩ��һ�׶����Ľӿڸ�RS����дmajor freeze��־ʱ�޸�
\_\_all\_core\_table
* \_\_all\_zone��
\_\_all\_zone���е�try_frozen_version��frozen_version���������ڱ�ʶmajor freeze�Ŀ�ʼ�ͽ��������赱ǰ��Ⱥ�Ķ���汾��ΪV�������¿���һ��major freezeǰ��������try_frozen_versionΪV+1�������������major freeze��frozen_versionҲ����ΪV+1���Ӷ���ʶ����major freeze���������⣬��RS�����ͼ�Ⱥ����ʱ��Ҳ��ͨ���Ƚ�try_frozen_version��frozen_version����ֵ�Ƿ�������жϵ�ǰ�Ƿ����δ��ɵ�major freeze��
* \_\_all\_server��
�°�major freeze��participantsΪobserver����ִ��major freezeʱ��Ҫ��ȡobserver�б���ȡobserve�б���ͨ����ȡ
\_\_all\_server����ɵġ�RSͨ����observer���������ά��\_\_all\_server���и�observer��״̬����observer��һ���ض���ʱ���ڶ�û����RS����������RS����Ϊ��observerʧ��������\_\_all\_server�����޸ĸ�observer��״̬��observer��RSʧȥ����ʱ��RS���޷�����observer��崻�״̬����RS��observer�䷢����������ġ�

###�°�major freeze��observer��Ҫ��
�������ᵽ���°��major freeze��������observer������partition���汾�����֣�Ϊ��֤����observer��ɺ��¿��������񲻻ᱻд��partition�ϰ汾��memtable�ϣ�observer�����������������е�һ����
* \_\_all\_server���е�����observer������alive״̬����������£���Ⱥ��ɶ�������е�partition�����Ը���observer�ϵĶ���״̬�ı仯������汾������
* \_\_all\_server���д��ڴ��ڷ�alive״̬observer��������÷�alive��observer�ϲ�����partition��Ҳ����observer��һ����serverʱ��Ҳ���Ա�֤major freeze����ȷ�ԡ�

###Root major freeze�Ļ�������
```nohighlight
 +-----+
 |begin|
 +-----+
   |
   v
 +----------------------------------+
 |__all_zone.try_frozen_vesion += 1 |
 +----------------------------------+
   |
   v
 +--------------------------------------+
 |write __all_core_table prepare record |<--------------------<-------------------+
 +--------------------------------------+                                         |
   |                                                                              |
   v                                                                              |
 +-------------------------------+     FAIL                                       |
 |send prepare request           |-------->--------+                              |
 +-------------------------------+                 |                              |
   |                                               |                              |
   | SUCCESS                                       |                              |           
   v                                               v                              |
 +-------------------------------------+  +-----------------------------------+   |
 |write __all_core_table commit record |  |write __all_core_table abort record|   |
 +-------------------------------------+  +-----------------------------------+   |
   |                                               |                              |
   | SUCCESS                                       |                              |
   v                                               v                              |
 +-------------------------------+        +-------------------------------+       |
 |send commit request            |        |send abort request             |       |
 +-------------------------------+        +-------------------------------+       |
   |                                               |                              |
   | SUCCESS                                       |                              |
   v                                               |                              |
 +------------------------------+                  +--------------->--------------+
 |__all_zone.frozen_vesion += 1 |
 +------------------------------+
   |
   v
 +---+
 |end|
 +---+

```
###major freeze���쳣����
major freeze�����д���RS�����ͼ�Ⱥ���������쳣���������쳣�����������һ����RS�����Σ���RS���κ���Ҫ��֮ǰδ��ɵ�major freeze�ƽ��ꡣ
* ��ǰ����״̬ΪINIT������������major freeze�Ļ������̽�major freeze�ƽ���ɡ�
* ��ǰ����״̬ΪCOMMIT_SUCCEED: RS�������observer����commit requestֱ���ɹ���
* ��ǰ����״̬ΪPREPARED\_SUCCEED����ʱ��Ⱥ���ܴ���ͣ����״̬��\_\_all\_zone��\_\_all\_server���ѡ���������д�뵽\_\_all\_root\_table����������major freeze�޷��������С�Ϊ���������������RS����ʱ�������ǰ�Ķ���״̬ΪPREPARED_SUCCEED,�����Ƚ�����״̬�޸�ΪINIT����ͨ��������������״̬ͬ������observer���ָ���ȺΪ��д����������major freeze�ƽ���ɡ�











