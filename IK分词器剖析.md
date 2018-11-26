# IK分词器剖析

中文分词器主要作用：

1.识文断字，在中文环境下识别所有词汇，词组，短语。

2.理解上下文含义输出符合语境的分词结果。



## 一、IK分词器组成

1.词典和词典树管理

2.文字匹配和原始分词结果管理

3.分词裁决器，歧义消除过程



## 二、词典和词典树管理

![字典截图1](.\图片\字典截图1.png)

### **2.1 词典分类**

1.主词典/扩展主词典  共计275909 + 39800 个词汇

2.量词  316个词汇

3.姓氏  131个词汇

4.后缀词(乡井亭党区厅县园塔家寺局) 37个

5.前缀词 25个。

6.停顿词  中文31 + 英文33。停顿词指文档中基本都包含的词，区分度低，不做分词处理(中文:也 了 仍 从 以 使 则 却 又 及 对 就 并 很)

**IK分词器词典在config目录下，可以支持自定义扩展，可以支持Http在线更新。**



### 2.2 词典树

​	词典树的组成是读取词典中个每个词条，拆解成单个字符后存放的。

![1ik分词](.\图片\ik分词.jpg)

词典树说明

1.词典树中每一个结点存放1个字符，如果\[京][梅]是2个结点

2.每个结点会存放子结点的集合，如\[东][城]2个结点存放在父节点\[京]的子结点集合中。

3.词典树中存在整词标记,如梅花、梅花鹿、京东、京东方，在末尾结点会有标记字段。

4.一次匹配可能可能是一次整词匹配+一次前缀匹配，如果"京东"或匹配京东和京东方的前缀。

5.根节点不存储字符，根节点的子结点集合存储词典文本中，所有词的第一个字。



### 2.3 认识词典树的结点类

**词典树结点DictSegment主要成员**

```java
class DictSegment implements Comparable<DictSegment>{
   //公用字典表，存储词典本文中的所有汉字，静态成员
private static final Map<Character , Character> charMap = new ConcurrentHashMap<Character , Character>(16 , 0.95f);
   //数组大小上限
   private static final int ARRAY_LENGTH_LIMIT = 3;
   //Map存储结构
   private Map<Character , DictSegment> childrenMap;
   //数组方式存储结构
   private DictSegment[] childrenArray;   
   //当前节点上存储的字符
   private Character nodeChar;
   //子节点存储的Segment数目
   //storeSize <=ARRAY_LENGTH_LIMIT，使用数组存储， storeSize >ARRAY_LENGTH_LIMIT ,则使用Map存储
   private int storeSize = 0;
   //当前DictSegment状态 ,默认0。1表示从根节点到当前节点的路径表示一个整词
   private int nodeState = 0; 
   }
```

### 2.4 词典加载和词典树的构造

​	逐行读取xxx.dic文件中的每行记录，一行对应一个词，写入词典树。

Dictionary::loadDictFile()方法，传入词典树根结点。

```java
private void loadMainDict() {
  	// 建立一个主词典的根实例,存入0无使用意义
  	_MainDict = new DictSegment((char) 0);
  	// 读取主词典文件
  	Path file = PathUtils.get(getDictRoot(), Dictionary.PATH_DIC_MAIN);
  	loadDictFile(_MainDict, file, false, "Main Dict");
  	// 加载扩展词典
  	this.loadExtDict();
  	// 加载远程自定义词库
  	this.loadRemoteExtDict();
}

private void loadDictFile(DictSegment dict, Path file) {
     InputStream is = new FileInputStream(file.toFile() 
      BufferedReader br = new BufferedReader(
            new InputStreamReader(is, "UTF-8"), 512);
      //逐行读取词典文本                                    
      String word = br.readLine();
      if (word != null) {
         for (; word != null; word = br.readLine()) {
            word = word.trim();
            if (word.isEmpty()) continue;
            //向词典树中写入一个词，入参dict是词典树的根结点
            dict.fillSegment(word.toCharArray());
         }
      }
}
```



**构造词典树结点**

1.flllSegment是递归调用函数，调用次数与词的字符数相同。

2.递归返回条件时写入到词的最后一个字符，此时设定整词标记并返回。

```java
private synchronized void fillSegment(char[] charArray , int begin , int length , int enabled){
   //获取字典表中的汉字对象
   Character beginChar = Character.valueOf(charArray[begin]);
   Character keyChar = charMap.get(beginChar);
   //字典中没有该字，则将其添加入字典
   if(keyChar == null){
      charMap.put(beginChar, beginChar);
      keyChar = beginChar;
   }
   //搜索当前子结点的存储，查询对应keyChar的keyChar，如果没有则创建
   DictSegment ds = lookforSegment(keyChar , enabled);
   if(ds != null){
      //处理keyChar对应的segment
      if(length > 1){
      //词还没有完全加入词典树时进行递归调用检查charArray下一个字符
         ds.fillSegment(charArray, begin + 1, length - 1 , enabled);
      }else if (length == 1){
         //已经是词元的最后一个char,设置当前节点状态为enabled，
         //enabled=1表明一个完整的词
         ds.nodeState = 1;
      }
   }

}
```



**词典树的查找与创建过程**

1.如果起始结点的子结点数量小于3个使用数组存储。

2.如果大于3个使用Map存储子结点Map<Character, DictSegment>。

ARRAY_LENGTH_LIMIT = 3

```java
private DictSegment lookforSegment(Character keyChar ,  int create){
   DictSegment ds = null;
   //在没有超过3个子结点的情况下使用ChildrenArray查找
   if(this.storeSize <= ARRAY_LENGTH_LIMIT){
      //获取数组容器，如果数组未创建则创建数组
      DictSegment[] segmentArray = getChildrenArray();         
      //搜寻数组，二分查找DictSegment实现了compareTo
      DictSegment keySegment = new DictSegment(keyChar);
      int position = Arrays.binarySearch(segmentArray, 0 , this.storeSize, keySegment);
      if(position >= 0){
         ds = segmentArray[position];
      }
      //遍历数组后没有找到对应的segment
      if(ds == null && create == 1){
         ds = keySegment;
         if(this.storeSize < ARRAY_LENGTH_LIMIT){
            //数组容量未满，使用数组存储
            segmentArray[this.storeSize] = ds;
            //当前树结点的子结点存储的字符数量+1
            this.storeSize++;
            Arrays.sort(segmentArray , 0 , this.storeSize);
         }else{
            //数组容量已满，切换Map存储
            //获取Map容器，如果Map未创建,则创建Map
            Map<Character , DictSegment> segmentMap = getChildrenMap();
            //将数组中的segment迁移到Map中
            migrate(segmentArray ,  segmentMap);
            //存储新的segment
            segmentMap.put(keyChar, ds);
            //当前树结点的子结点存储的字符数量+1
            this.storeSize++;
            //释放当前的数组成员引用
            this.childrenArray = null;
         }
      }        
   }else{
      //获取Map容器，如果Map未创建,则创建Map
      Map<Character , DictSegment> segmentMap = getChildrenMap();
      //搜索Map
      ds = (DictSegment)segmentMap.get(keyChar);
      if(ds == null && create == 1){
         //构造新的segment
         ds = new DictSegment(keyChar);
         segmentMap.put(keyChar , ds);
         //当前节点存储segment数目+1
         this.storeSize ++;
      }
   }
   return ds; //返回查找到或构造的子结点做递归调用的起始结点
}

public int compareTo(DictSegment o) {
  //对当前节点存储的char进行比较
  return this.nodeChar.compareTo(o.nodeChar);
}
```



## 三、词和词的匹配

### 3.1 输入文本缓存上下文

```java
class AnalyzeContext {
   //默认缓冲区
   private static final int BUFF_SIZE = 4096;
   //缓冲区耗尽的临界值(未处理字符小于100 且 当前缓冲区满载时进行下一次读取)
   private static final int BUFF_EXHAUST_CRITICAL = 100;  
   //缓冲区对象
   private char[] segmentBuff;
   //字符类型数组
   private int[] charTypes;
   //记录Reader内已分析的字串总长度
   //在多次读入缓冲区时，该变量累计当前的segmentBuff相对于reader起始位置的位移
   //1.第一次读取100字,buffOffset=0,第二次读取时buffOffset = 100
   private int buffOffset;    
   //当前缓冲区位置指针(指向当前第一个未处理字符)
   private int cursor;
   //最近一次读入的,待处理的字串长度
   private int available;
   //原始分词结果集合，未经歧义处理的结果集
   private QuickSortSet orgLexemes;    
   //最终分词结果集
   private LinkedList<Lexeme> results;
}
```

### 3.2 词元和匹配结果对象

词元指在词典树中成功匹配一个字符串。

```java
public class Lexeme implements Comparable<Lexeme>{
   //词元所在AnalyzeContext 的 buffOffset 值
   private int offset;
    //词元在segmentBuff中的起始偏移值
    private int begin;
    //词元的长度
    private int length;
    //词元文本
    private String lexemeText;
    //词元是有大小排序的规则是
    //1.起始偏移较小的词较大 2.起始偏移相当时，长度大的词较大
  	public int compareTo(Lexeme other) {
		//起始位置优先
        if(this.begin < other.getBegin()){
            return -1;
        }else if(this.begin == other.getBegin()){
        	//词元长度优先
        	if(this.length > other.getLength()){
        		return -1;
        	}else if(this.length == other.getLength()){
        		return 0;
        	}else {//this.length < other.getLength()
        		return 1;
        	}
        }else{//this.begin > other.getBegin()
        	return 1;
        }
	}
}
```

词元(Lexeme)需要比较的原因是，它需要存入双向链表对象QuickSortSet并做降序排列。

```java
class QuickSortSet {
   //链表头
   private Cell head;
   //链表尾
   private Cell tail;
   //链表的实际大小
   private int size;
   QuickSortSet(){
      this.size = 0;
   }
   //内部类Cell实际存入的一个词元对象lexeme;
   class Cell implements Comparable<Cell>{
		private Cell prev;
		private Cell next;
		private Lexeme lexeme;
		Cell(Lexeme lexeme){
			this.lexeme = lexeme;
		}
}
```

**降序排列存放过程**

QuickSortSet::addLexeme()

```java
boolean addLexeme(Lexeme lexeme){
   Cell newCell = new Cell(lexeme); 
   if(this.size == 0){
      this.head = newCell;
      this.tail = newCell;
      this.size++;
      return true;
   }else{
     //词元与尾部词元相同，不放入集合
      if(this.tail.compareTo(newCell) == 0){
         return false;
      }else if(this.tail.compareTo(newCell) < 0){
        //newCell比尾词元小，词元接入链表尾部
         this.tail.next = newCell;
         newCell.prev = this.tail;
         this.tail = newCell;
         this.size++;
         return true;
      }else if(this.head.compareTo(newCell) > 0){
        //newCell比头词元大时，词元接入链表头部
         this.head.prev = newCell;
         newCell.next = this.head;
         this.head = newCell;
         this.size++;
         return true;
      }else{             
         //从链表尾部最小的词开始，逆序向头部查找一个合适的位置
         Cell index = this.tail;
         while(index != null && index.compareTo(newCell) > 0){
         //遍历词元比新词元小（index的begin靠后或length较小）
            index = index.prev;
         }
         //词元与集合中的词元重复，不放入集合
         if(index.compareTo(newCell) == 0){
            return false; 
         }
         //找到一个比newCell大的词元，插入链表中该index词元后
         else if(index.compareTo(newCell) < 0){
            newCell.prev = index;
            newCell.next = index.next;
            index.next.prev = newCell;
            index.next = newCell;
            this.size++;
            return true;               
         }
      }
   }
   return false;
}
```



**降序排列举例分词“京东物流”  链表中的结果“京东物流 --- 京东 --- 物流”。**

**匹配结果Hit**

```java
public class Hit {
   //Hit不匹配
   private static final int UNMATCH = 0x00000000;
   //Hit完全匹配
   private static final int MATCH = 0x00000001;
   //Hit前缀匹配
   private static final int PREFIX = 0x00000010;
   //该HIT当前状态，初始时未匹配
   private int hitState = UNMATCH;
   //记录词典匹配过程中，当前匹配到的词典树结点，用做下次匹配的起始结点
   private DictSegment matchedDictSegment; 
   //词段开始位置
   private int begin;
   //词段的结束位置
   private int end;
}
```



### 3.3 分词匹配过程

#### 3.3.1 分析处理步骤

```java
public synchronized Lexeme next()throws IOException{
   Lexeme l = null;
   while((l = context.getNextLexeme()) == null ){
      // 从reader中读取数据，填充buffer
      int available = context.fillBuffer(this.input);
      //初始化指针
       context.initCursor();
       do{
             //遍历子分词器
             for(ISegmenter segmenter : segmenters){
                segmenter.analyze(context);
             }
            //向前移动指针
       }while(context.moveCursor());
      //对分词进行歧义处理
      this.arbitrator.process(context, configuration.isUseSmart());
      //将分词结果输出到结果集，并处理未切分的单个CJK字符
      context.outputToResult();
      //记录本次分词的缓冲区位移
      context.markBufferOffset();          
   }
   return l;
}
```



#### 3.3.2 分词匹配

DictSegment::match()返回一个词/字在词典树的匹配结果

1.整词匹配

2.前缀匹配

3.整词和前缀匹配

4.不匹配(词典树中不存在输入词)

tips: match方法的length通常出入的是1即每次匹配1个字符

```java
Hit match(char[] charArray , int begin , int length , Hit searchHit){
		if(searchHit == null){
			//如果hit为空，说明上一次没有暂存的起始结点需要继续匹配
			searchHit= new Hit();
			//设置hit的其实文本位置
			searchHit.setBegin(begin);
		}
		//设置hit的当前处理位置
		searchHit.setEnd(begin);
        Character keyChar = Character.valueOf(charArray[begin]);
        //查找结果，初始为空
		DictSegment ds = null;
		//以下2个子结点容器只有1个不为空
	DictSegment[] segmentArray = this.childrenArray;
	Map<Character , DictSegment> segmentMap = this.childrenMap;	
		//STEP1 如果子结点数量小于3，此时在数组中查找字符
	if(segmentArray != null){
	DictSegment keySegment = new DictSegment(keyChar);
          //在数组中查找二分查找
	int position = Arrays.binarySearch(segmentArray, 0 , this.storeSize , keySegment);
	if(position >= 0){
           //在数组中找到的情况，写入到结果对象
			ds = segmentArray[position];
		}else if(segmentMap != null){
			//在map中查找并写入结果
			ds = (DictSegment)segmentMap.get(keyChar);
		}
		//STEP2 找到DictSegment，判断词的匹配状态，是否继续递归，还是返回结果
		if(ds != null){			
			if(length > 1){
		//词未匹配完，继续往下搜索，反应一个输入词在词典中的匹配情况
    return ds.match(charArray, begin + 1 , length - 1 , searchHit);
			}
             else if (length == 1){
				//搜索最后一个char
				if(ds.nodeState == 1){
					//添加HIT状态为完全匹配
					searchHit.setMatch();
				}
           //上面一个if可能已经写入完全匹配此时前缀匹配和完全匹配共存
				if(ds.hasNextNode()){
					//该结点还有子结点添加HIT状态为前缀匹配
					searchHit.setPrefix();
				//重要，记录当前结点位置的DictSegment作为下一次起点
					searchHit.setMatchedDictSegment(ds);
				}
				return searchHit;
			}
		}
		//STEP3 没有找到DictSegment， 将HIT设置为不匹配
		return searchHit;		
	}
```



#### 3.3.3 分词器对象

```java
class CJKSegmenter implements ISegmenter {  
   //子分词器标签
   static final String SEGMENTER_NAME = "CJK_SEGMENTER";
   //待处理的分词hit队列
   private List<Hit> tmpHits;
   CJKSegmenter(){
      this.tmpHits = new LinkedList<Hit>();
   }
```

分词主方法

CJKSegmenter::analyze主要过程使用下面实例解释。

prepare:词典树中有词"京东物流--京东--物流" 3个词入下图

![分词举例](D:.\图片\分词举例.gif)

当输入“京东物流”4个字的文本，匹配的过程是

1.初始tmpHits为空，之前前缀匹配的临时结点，对输入文本做单字匹配。

2.京---匹配 "京"所在的DictSegment --->child[东] ，前缀匹配，tmpHits存入

{dict[京 --- [东]]}



3.东---- 拿出tmpHits中的结点逐个处理，此时只有1个 dict[京 --- [东]，此时东在其子结点拿出 dict[东---[物]]，发现该结点的nodeState=1，因为"京东"是整词，同时写入match，prefix中匹配标记，做一下3步骤

3.1.tmpHits更新His元素的dict[京 --- [东]]) 为dict[东 --- [物]]

3.2 将京东一次创建一个Lexeme(京东)对象，写入到AnaylzeContext中的QuickSortSet

3.3 将"东"字做首字匹配 发现没有东开始的词，不做处理。

4.物 --- 拿出tmpHits中的结点逐个处理，dict[东 --- [物]])，匹配到 dict[物----[流]]，此时为前缀匹配，tmpHits更新Hit元素，dict[东 --- [物]])，为dict[物----[流]]结点。

4.1 将"物"字做首字匹配，发现dict[物---[流]]，此时为前缀匹配，追加到tmpHits

dict[物---[流]]，  dict[物---[流]] ，两个子结点分别来自不同的词，所以是不同的结点对象.



5.流 ---- 拿出tmpHits中的结点逐个处理，发现2个结点

dict[物---[流]]，  dict[物---[流]] 都能做整词匹配，输出2个词到AnalyzeContext的QuickSortSet对象:Lexeme(京东物流) ,Lexeme(物流)。清除tmpHits，结束。

注意:因为链表的写入时有序的此时顺序为"京东物流"--"京东"--"物流"。

附代码片段

```java
if(!this.tmpHits.isEmpty()){
      //处理词段队列
    Hit[] tmpArray = this.tmpHits.toArray(new Hit[this.tmpHits.size()]);
      for(Hit hit : tmpArray){
         hit = Dictionary.getSingleton().matchWithHit(context.getSegmentBuff(), context.getCursor() , hit);
         if(hit.isMatch()){
            //输出当前的词
            Lexeme newLexeme = new Lexeme(context.getBufferOffset() , hit.getBegin() , context.getCursor() - hit.getBegin() + 1 , Lexeme.TYPE_CNWORD);
            context.addLexeme(newLexeme);
            if(!hit.isPrefix()){//不是词前缀，hit不需要继续匹配，移除
               this.tmpHits.remove(hit);
            }
            
         }else if(hit.isUnmatch()){
            //hit不是词，移除
            this.tmpHits.remove(hit);
         }              
      }
   }        
   
   //*********************************
   //再对当前指针位置的字符进行单字匹配
   Hit singleCharHit = Dictionary.getSingleton().matchInMainDict(context.getSegmentBuff(), context.getCursor(), 1);
   if(singleCharHit.isMatch()){
      //输出当前的词
      Lexeme newLexeme = new Lexeme(context.getBufferOffset() , context.getCursor() , 1 , Lexeme.TYPE_CNWORD);
      context.addLexeme(newLexeme);
      //同时也是词前缀
      if(singleCharHit.isPrefix()){
         //前缀匹配则放入hit列表
         this.tmpHits.add(singleCharHit);
      }
   }else if(singleCharHit.isPrefix()){//首字为词前缀
      //前缀匹配则放入hit列表
      this.tmpHits.add(singleCharHit);
   }
  
}else{
   //遇到CHAR_USELESS字符
   //清空队列
   this.tmpHits.clear();
}

//判断缓冲区是否已经读完
if(context.isBufferConsumed()){
   //清空队列
   this.tmpHits.clear();
}

```



## 四、歧义处理器

**IKArbitrator**用户处理Context中的原始分词结果并输出到Context的result成员。

如果没有smart模式，IK分词器会输出所有的分词结果。

如:中华人民。分词得到"中华"、"华人"、"人民"。

在smart模式下IK会分析词元的位置重叠关系输出"位置"互补交叠且尽量贴合中国人理解的结果。

如:中华人民。分词得到"中华"、"人民"。

### 4.1 词链

```java
class LexemePath extends QuickSortSet implements Comparable<LexemePath>{
   //起始位置
   private int pathBegin;
   //结束
   private int pathEnd;
   //词元链的有效字符长度
   private int payloadLength;
   
   LexemePath(){
      this.pathBegin = -1;
      this.pathEnd = -1;
      this.payloadLength = 0;
   }
}
```

词链用于管理多个[词元]组成的[链表]

交叠检查:即检查一个词元与当前词链管理的多个词元之间的交叠关系。

```java
boolean checkCross(Lexeme lexeme){
   return
(lexeme.getBegin() >= this.pathBegin && lexeme.getBegin() < this.pathEnd)
  || 
(this.pathBegin >= lexeme.getBegin() && this.pathBegin < lexeme.getBegin()+ lexeme.getLength());
}
```

加入相交的词元

```java
boolean addCrossLexeme(Lexeme lexeme){
   if(this.isEmpty()){
      //空词链可以加入任何词元,判断相交成功
      this.addLexeme(lexeme);
      this.pathBegin = lexeme.getBegin();
      this.pathEnd = lexeme.getBegin() + lexeme.getLength();
      this.payloadLength += lexeme.getLength();
      return true;
   }else if(this.checkCross(lexeme)){
      this.addLexeme(lexeme);
      if(lexeme.getBegin() + lexeme.getLength() > this.pathEnd){
         this.pathEnd = lexeme.getBegin() + lexeme.getLength();
      }
      this.payloadLength = this.pathEnd - this.pathBegin;
      return true;
      
   }else{
      return  false; 
   }
}
```

加入不相交词元

```java
boolean addNotCrossLexeme(Lexeme lexeme){
   if(this.isEmpty()){
     //空词链可以加入任何词元，判断不相交成功
      this.addLexeme(lexeme);
      this.pathBegin = lexeme.getBegin();
      this.pathEnd = lexeme.getBegin() + lexeme.getLength();
      this.payloadLength += lexeme.getLength();
      return true;
   }else if(this.checkCross(lexeme)){
      return  false;
   }else{
      this.addLexeme(lexeme);
      this.payloadLength += lexeme.getLength();
      Lexeme head = this.peekFirst();
      this.pathBegin = head.getBegin();
      Lexeme tail = this.peekLast();
      this.pathEnd = tail.getBegin() + tail.getLength();
      return true;
   }
}
```

### 4.2 原始词元结果集输出

IKArbitrator::process()

1.中华/华人/人民。3个词元依次交叠。

说明：下面代码第9行会全部成功，在33行将crossPath保存的3个词输出到结果集

2.京东/物流/高效。3个词元互不交叠。

说明：下面代码第9行第一次会成功，后面循环会失败，进入14行出一次写入\[京东][物流]，25行会创建新的词链，第33行写入[高效]。

```java
void process(AnalyzeContext context , boolean useSmart){
   //原始分词结果组成的双向链表，内部次有大小排序
   //京东-物流-高效  中华-华人-人民
   QuickSortSet orgLexemes = context.getOrgLexemes();
   //取第一个词元
   Lexeme orgLexeme = orgLexemes.pollFirst();
  
   LexemePath crossPath = new LexemePath();
   while(orgLexeme != null){
      if(!crossPath.addCrossLexeme(orgLexeme)){
         //找到与crossPath不相交的下一个crossPath 
         if(crossPath.size() == 1 || !useSmart){
            //crossPath没有歧义 或者 不做歧义处理
            //直接输出当前crossPath
            context.addLexemePath(crossPath);
         }else{
            //对当前的crossPath进行歧义处理
            QuickSortSet.Cell headCell = crossPath.getHead();
            LexemePath judgeResult = this.judge(headCell, crossPath.getPathLength());
            //输出歧义处理结果judgeResult
            context.addLexemePath(judgeResult);
         }
         //把orgLexeme加入新的crossPath中
         crossPath = new LexemePath();
         crossPath.addCrossLexeme(orgLexeme);
      }
      //分析下个词元
      orgLexeme = orgLexemes.pollFirst();
   }
   //处理最后的path
   if(crossPath.size() == 1 || !useSmart){
      //crossPath没有歧义 或者 不做歧义处理
      //直接输出当前crossPath
      context.addLexemePath(crossPath);
   }else{
      //对当前的crossPath进行歧义处理
      QuickSortSet.Cell headCell = crossPath.getHead();
      LexemePath judgeResult = this.judge(headCell, crossPath.getPathLength());
      //输出歧义处理结果judgeResult
      context.addLexemePath(judgeResult);
   }
}
```

### 4.3 智能模式结果集输出

prepare：京东物流 - 京东 - 物流国际化 - 物流

输入：京东物流国际化 (发展迅速)。

输出：京东物流/国际化 (发展迅速)。？

输出：京东/物流国际化 (发展迅速)。？



```java
QuickSortSet.Cell headCell = crossPath.getHead();
//处理有相交的词元链，输出一个不相交的词链且便于理解
LexemePath judgeResult = this.judge(headCell);
//输出歧义处理结果judgeResult
context.addLexemePath(judgeResult);
```

裁决过程

**京东物流 - 京东 - 物流国际化 - 物流**

```java
//回滚词元链从后往前删除option中与c相交的词 文本"京东物流"为例
//option 京东--物流  l= 物流  结果option只剩下京东
//option 京东物流  l= 物流    结果option为空
private void backPath(Lexeme l, LexemePath option){
  while(option.checkCross(l)){
    option.removeTail();
  }
}

//京东物流 - 京东 - 物流国际化 - 物流 4个词的结果为例
private Stack<QuickSortSet.Cell> forwardPath(QuickSortSet.Cell lexemeCell , LexemePath option){
   //发生冲突的Lexeme栈
   Stack<QuickSortSet.Cell> conflictStack = new Stack<QuickSortSet.Cell>();
   QuickSortSet.Cell c = lexemeCell;
   //迭代遍历Lexeme链表
   while(c != null && c.getLexeme() != null){
      //option为空时总会成功 option会写入lexemeCell传入的词元
      //！！！京东 与 物流国际化不相交 可以同时存入option
      if(!option.addNotCrossLexeme(c.getLexeme())){
         //词元交叉，添加失败则加入lexemeStack栈
         conflictStack.push(c);
      }
      c = c.getNext();
   }
   return conflictStack; 
}

//京东物流 - 京东 - 物流国际化 - 物流 4个词的结果为例
private LexemePath judge(QuickSortSet.Cell lexemeCell , int fullTextLength){
   //候选路径集合
   TreeSet<LexemePath> pathOptions = new TreeSet<LexemePath>();
   //候选结果路径
   LexemePath option = new LexemePath();
   //对crossPath进行一次遍历,同时返回本次遍历中有冲突的Lexeme栈
   Stack<QuickSortSet.Cell> lexemeStack = this.forwardPath(lexemeCell , option); 
    //第一次返回的冲突栈 
    //【物流】
    //【物流国际化】
    //【京东】
   //当前词元链可能并非最理想的，加入候选路径集合
   //第一次返回option = 京东物流 写入treeSet
   pathOptions.add(option.copy());
   //存在歧义词，处理
   QuickSortSet.Cell c = null;
   while(!lexemeStack.isEmpty()){
      // c = 物流  物流国际化 京东
      c = lexemeStack.pop();
      //option = 京东物流 c = 物流
      //回滚词元链从后往前删除option中与c相交的词
      this.backPath(c.getLexeme() , option);
      //此时option为空因为 京东物流 与 物流有交叠backPath把option清空了
      this.forwardPath(c , option);
      //option = 物流 写入TreeSet
      pathOptions.add(option.copy());
   }
   //返回集合中的最优方案
   return pathOptions.first();
}
```

**京东物流 - 京东 - 物流国际化 - 物流**

pathOptions = 京东物流

pathOptions = 京东物流 | 物流 |

pathOptions = 京东物流 | 物流 | 物流国际化

pathOptions = 京东物流 | 物流 | 物流国际化 | 京东-物流国际化

**return pathOptions.first();** 出输出的结果是？

```java
public int compareTo(LexemePath o) {
   //比较有效文本长度 长度越长越好
   if(this.payloadLength > o.payloadLength){
      return -1;
   }else if(this.payloadLength < o.payloadLength){
      return 1;
   }else{
      //比较词元个数，越少越好,优先取长词
      if(this.size() < o.size()){
         return -1;
      }else if (this.size() > o.size()){
         return 1;
      }else{
         //路径跨度越大越好,词元覆盖的范围约广越好
         if(this.getPathLength() >  o.getPathLength()){
            return -1;
         }else if(this.getPathLength() <  o.getPathLength()){
            return 1;
         }else {
            //根据统计学结论，逆向切分概率高于正向切分，因此位置越靠后的优先
            if(this.pathEnd > o.pathEnd){
               return -1;
            }else if(pathEnd < o.pathEnd){
               return 1;
            }else{
               //词长越平均越好 4x4x4 > 3x4x5 weight即长度相乘
               if(this.getXWeight() > o.getXWeight()){
                  return -1;
               }else if(this.getXWeight() < o.getXWeight()){
                  return 1;
               }else {
                  //词元位置权重比较，权重大获胜
                  if(this.getPWeight() > o.getPWeight()){
                     return -1;
                  }else if(this.getPWeight() < o.getPWeight()){
                     return 1;
                  }
                  
               }
            }
         }
      }
   }
   return 0;
}
```