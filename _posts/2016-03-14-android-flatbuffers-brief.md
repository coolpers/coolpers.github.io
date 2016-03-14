
---
layout: post
title:  "android FlatBuffers剖析"
date:   2016-03-14
categories: android | flatbuffers
tags: android flatbuffers
---

# android FlatBuffers剖析 #


## 概述 ##
FlatBuffers是google最新针对游戏开发退出的高性能的跨平台序列化工具，目前已经支持C++, C#, Go, Java, JavaScript, PHP, and Python (C和Ruby正在支持中)，相对于json和Protocol Buffers，FlatBuffers在序列化和反序列化方面表现更为优异，而且需要的资源更少，更适合大部分移动应用的使用场景。

## FlatBuffers的特点 ##
除了高性能和低内存消耗的特点，FlatBuffers还有其他一些优势，官方的总结说明如下：

* 不用解析也能对数据进行访问  
FlatBuffers使用二进制结构化的方法来存储数据，可以不用先对整段数据进行解析就可以直接在二进制结构中访问到指定成员的信息。并且FlatBuffers还支持存储数据结构的前后兼容。
* 内存占用小，运行效率高  
FlatBuffers使用ByteBuffer进行数据存储，在FlatBuffers的序列化和反序列化过程中不会额外占用其他内存空间，FlatBuffers读取数据的速度和直接从内存读取原始数据相当，仅仅多了一个相对寻址的耗时。同时，数据存储在ByteBuffer中还能够比较容易地支持内存映射（mmap）和流式读写，进一步降低对内存的消耗。
* 灵活  
FlatBuffers支持选择性地写入数据成员，这不仅为某一个数据结构在应用的不同版本之间提供了兼容性，同时还能使程序员灵活地选择是否写入某些字段及灵活地设计传输的数据结构。
* 体积小，集成成本低  
项目中集成FlatBuffers只需要很少的额外代码。
* 强类型约束  
如果数据结构不符合FlatBufferss文件的描述，那么会在编译期间就发现问题，而不会在运行时才暴露问题。
* 使用方便  
可以兼容json等其他格式的解析。
* 跨平台

FlatBuffers和Protocol Buffers是比较相似的，但是FlatBuffers不需要在读取成员变量之前必须将数据完全解析成对象，因为它所有信息的读取都是在对应的ByteBuffer中进行的，少了这些解析时必须为对象和成员变量分配的内存空间，就降低了解析过程中的内存消耗。json相对于FlatBuffers来说可读性更好，但是缺点也是明显的，那就是它的性能太低了，这点可以参见FlatBuffers的[benchmarks](http://google.github.io/flatbuffers/flatbuffers_benchmarks.html)。其他FlatBuffers的优势可以看[white paper](http://google.github.io/flatbuffers/flatbuffers_white_paper.html)。

## FlatBuffers的基本使用 ##
由于本文仅仅介绍在android应用中使用FlatBuffers的方法，因此基本使用方法也只针对java语言进行介绍，其他语言的使用介绍请参看[官方介绍](http://google.github.io/flatbuffers/index.html#flatbuffers_overview)。

### 下载FlatBuffers源码并编译 ###
FlatBuffersS的源码包含flatc的源代码及支持的各种语言需要依赖的代码，目前托管在github上面（[这里](http://github.com/google/flatbuffers)）。下载完成后首先需要将源码中的flatc源码编译成自己所用平台上的flatc工具，flatc源码支持使用visual studio和xcode进行编译，也支持使用cmake进行跨平台编译，在mac上使用cmake进行编译的方法可以参看[这里](http://blog.csdn.net/yxz329130952/article/details/50706369)。在编译得到flatc后，就可以先将源码目录下自己所用语言（这里是java/com/google/flatbuffers）目录引入到自己的工程目录下就可以进行下一步工作了。


### 编写schema文件 ###
FlatBuffers需要一个用IDL语言描述的schema文件来定义传输数据的结构，IDL是一种类似于c语言的接口定义语言，它支持bool、short、float和double几种基本数据结构及数组、字符串、Struct和Table几种复杂类型。关于如何使用IDL来编写schema文件可以参看[这里](http://google.github.io/flatbuffers/flatbuffers_guide_writing_schema.html)，此处就不做过多的描述。这里为了方面还是以官方demo的schema文件（*monster.FlatBufferss*）为例，相应的代码如下：

	// Example IDL file for our monster's schema.
	namespace MyGame.Sample;
	enum Color:byte { Red = 0, Green, Blue = 2 }
	union Equipment { Weapon } // Optionally add more tables.
	struct Vec3 {
	  x:float;
	  y:float;
	  z:float;
	}
	table Monster {
	  pos:Vec3; // Struct.
	  mana:short = 150;
	  hp:short = 100;
	  name:string;
	  friendly:bool = false (deprecated);
	  inventory:[ubyte];  // Vector of scalars.
	  color:Color = Blue; // Enum.
	  weapons:[Weapon];   // Vector of tables.
	  equipped:Equipment; // Union.
	}
	table Weapon {
	  name:string;
	  damage:short;
	}
	root_type Monster;

### 编译schema文件 ###
编写完schema文件后，就需要使用flatc将其转换成对应语言所对应的类，这里使用：

	flatc --java samples/monster.FlatBufferss

将schema文件转换成了如下几个java文件：

![03_flac编译生成的文件](/assets/posts/2016-03-14-android-flatbuffers/03_flac编译生成的文件.png)

这几个文件就是要读写这个schema文件对应的FlatBuffers数据结构所需要依赖的类。flatc会按照一套严格的标准来完成转换的工作，打开生成的这些文件可以看到，这几个类的实现有很多写死的常量，例如成员变量的索引和数组的大小等，这也就说明，一旦schema文件编写完成，也就相当于确定了相应数据的存储结构。flatc还支持很多参数来完成不同的工作，更多高级特性请参看[这里](http://google.github.io/flatbuffers/flatbuffers_guide_using_schema_compiler.html)

### 使用 ###
将上一步使用flatc编译生成的java文件引入工程，这些文件就相当于是schema中定义的数据结构的java封装，我们可以很方便地通过这些类完成数据的序列化和反序列化工作。并且只要反序列化时使用的schema和序列化时使用的schema一致，那么一定可以完整地还原序列化时的数据，而和序列化和反序列化时使用的语言无关，这一点是通过FlatBuffers构建二进制数据的规则来保证的，稍后会具体分析这一点。引入编译生成的java类后，可以通过如下代码简单地进行demo测试。


	    private static void testFlatBuffers() {
        
        /*************************** 序列化 ************************/
        
        /**
         *  FlatBufferBuilder是FlatBuffers进行序列化和反序列化的关键类，这个类内部定义了各种数据类型进行序列化和反序列化的规则，
         *  并且封装了一系列数据结构序列化和反序列化时需要进行的操作。
         */
        FlatBufferBuilder builder = new FlatBufferBuilder();    

        // 创建一个字符串名称，返回的是字符串在底层ByteBuffer中的偏移
        int monsterName = builder.createString("软泥麦塔");     

        // 通过生成的Monster类存储一个数组信息，其实最终数据还是通过FlatBuilder来完成存储的
        byte[] treasure = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };     
        int inv = Monster.createInventoryVector(builder, treasure);     

        /**
         * 根据Weapon类封装的方法创建武器信息，之所以Weapon类要封装这个操作是因为Weapon要根据
         * schema对自身的定义写死一些常量，例如Weapon的成员个数以及各个成员的索引。
         */
        int weapon1Name = builder.createString("axe");
        short axeDamage = 50;
        int axe = Weapon.createWeapon(builder, weapon1Name, axeDamage);

        int weapon2Name = builder.createString("锈刀");
        short swordDamage = 100;
        int sword = Weapon.createWeapon(builder, weapon2Name, swordDamage);
        
        // Pass the `weaps` array into the `createWeaponsVector()` method to create a FlatBuffer vector.
        int[] weaps = new int[2];
        weaps[0] = sword;
        weaps[1] = axe;
        int weapons = Monster.createWeaponsVector(builder, weaps);
        
        /**
         * 上面的那些类型是不能内联存储到Monster中的，因为它们属于schema中的复杂类型，需要先单独使用FlatBuilder在底层的
         * ByteBuffer中存储，然后将各个类型的偏移保存起来构建Monster信息。
         */
        Monster.startMonster(builder);      // 准备构建Monster信息
        Monster.addName(builder, monsterName);  // 使用之前得到的偏移构建怪兽名称
        /**
         * Vec3是Struct类型，Struct类型是用来保存固定不变的数据的，一旦规定变不可更改（不能增加也不能减少），
         * 并且FlatBuffers为了提高数据访问效率，Struct的数据访问没有适用相对寻址，而是适用了直接寻址，因此它的数据只能内联存储。
         */
        int postion = Vec3.createVec3(builder, 1, 2, 3);    // vector必须要内联存储
        Monster.addPos(builder, postion);
        Monster.addColor(builder, Color.Blue);  // 简单数据也进行内联存储
        Monster.addHp(builder, (short) 700);
        Monster.addMana(builder, (short) 10);
        Monster.addInventory(builder, inv);
        Monster.addWeapons(builder, weapons);
        Monster.addEquippedType(builder, Equipment.Weapon);
        Monster.addEquipped(builder, axe);
        int rootMonster = Monster.endMonster(builder); // monster 构造结束

        builder.finish(rootMonster); // 所有数据写入完毕，写入FlatBuffers root_table的位置

        ByteBuffer data = builder.dataBuffer();     //得到底层存储二进制数据的ByteBuffer
        
        // 写入数据到文件
        File file = new File("flatbuffersFile.txt");
        FileOutputStream out = null;
        FileChannel channel = null;
        try {
            out = new FileOutputStream(file);
            channel = out.getChannel();
            while (data.hasRemaining()) {
                try {
                    channel.write(data);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } finally {
            try {
                if (null != out) {
                    out.close();
                }
                
                if (null != channel) {
                    channel.close();
                }
            } catch (Exception e2) {
            }
            
        }
        
        /*************************** 反序列化 ************************/
        
        // 从文件中读取数据
        FileInputStream fis = null;
        FileChannel readChannel = null;
        try {
            fis = new FileInputStream(file);
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
            readChannel = fis.getChannel();
            int readbytes = 0;
            while ((readbytes = readChannel.read(byteBuffer)) != -1)  {
                System.out.println("读到 " + readbytes + " 个数据");
            }
            byteBuffer.flip();  // position回绕，准备从ByteBuffer中读取数据
            
            Monster monster = Monster.getRootAsMonster(byteBuffer);     // 找到root_table的位置，即数据的入口
            
            // 测试读取的数据是否和写入的数据一致。
            System.out.println("monster name: " + monster.name() + " hp: " + monster.hp());
        } catch (Exception e) {
            e.printStackTrace();
            try {
                if (null != readChannel) {
                    readChannel.close();
                }
                if (null != fis) {
                    fis.close();
                }
            } catch (Exception e2) {
                e2.printStackTrace();
            }
        }
    }

相关的操作都已经在注释中写明白了。总的来说，FlatBuffers组织数据的格式和java class文件的格式有点类似，一些复杂类型的成员都先按照约定的格式先写入底层的ByteBuffer，这类似于java class中的常量区，然后使用这些常量的偏移来构建Table，这就相当于java class文件中的“表”。这样的结构决定了一些复杂类型的成员都是使用相对寻址进行数据访问的，即先从Table中取到成员常量的偏移，然后根据这个偏移再去常量真正存储的地址去取真实数据。但是Strcut类型的数据算是一个例外，FlatBuffers规定Struct类型用于存储那些约定成俗、永不改变的数据，这种类型的数据结构一旦确定便永远不会改变，在这个规定之下，为了提高数据访问速度，FlatBuffers单独对Struct使用了直接寻址的方式，这也要求了其数据必须进行内联存储。


## FlatBuffers数据存储结构 ##
虽然通过使用flatc对idl文件进行编译后，会自动生成我们定义的数据结构的java类，并且在使用过程中我们也不用关心数据的存储细节。但是如果你想要了解为什么FlatBuffers会如此高效，那么首先就不得不清楚FlatBuffers中的各种不同类型的数据结构是如何存储的。  
FlatBuffers底层使用了java的ByteBuffer进行数据存储，ByteBuffer可以算是java NIO体系中的重要成员，很多jvm单独为它从heap中划分了一块存储区域进行数据存储，这样就避免了java数据到native层的传输需要经过java heap到native heap的数据拷贝过程，从而提高了数据读写的效率。但是ByteBuffer是针对直接进行数据存取操作的，虽然它提供了诸如asIntBuffer等方法来构造包装类以便针对int等类型的数据进行读取，但是毕竟FlatBuffers存储的一般并不是单一的数据类型，因此如果让用户来直接操作底层的ByteBuffer的话还是非常麻烦的。幸运的是FlatBuffersBuilder已经为我们封装了很多操作。
  
### FlatBuffers对ByteBuffer的基本使用原则 ###
在后面详细介绍各种数据存储结构之前先说一下FlatBuffers是按照什么规则来使用ByteBuffer的，总的说来就是以下两点：

* 小端模式  
FlatBuffers对各种基本数据的存储都是按照小端模式来进行的，因为这种模式目前和大部分处理器的存储模式是一致的，可以加快数据读写的数据。这一点FlatBuffers一般是在初始化ByteBuffer的时候调用*ByteBuffer.order(ByteOrder.LITTLE_ENDIAN)*来实现的。
* 写入数据方向和读取数据方向不同  
和一般向ByteBuffer写入数据的习惯不同，FlatBuffers向ByteBuffer中写入数据的顺序是从ByteBuffer的尾部向头部填充，由于这种增长方向和ByteBuffer默认的增长方向不同，因此FlatBuffers在向ByteBuffer中写入数据的时候就不能依赖ByteBuffer的position来标记有效数据位置，而是自己维护了一个space变量来指明有效数据的位置，在分析FlatBuffersBuilder的时候要特别注意这个变量的增长特点。但是，和数据的写入方向不同的是，FlatBuffers从ByteBuffer中解析数据的时候又是按照ByteBuffer正常的顺序来进行的。FlatBuffers这样组织数据存储的好处是，在从左到右解析数据的时候，能够保证最先读取到的就是整个ByteBuffer的概要信息（例如Table类型的vtable字段），方便解析。如下图所示：

![01_FlatBuffers中ByteBuffer数据增长方向](/assets/posts/2016-03-14-android-flatbuffers/01_FlatBuffers中ByteBuffer数据增长方向.png)

但是为什么FlatBuffers要费劲地在写的时候将数据做逆向增长？这个我也确实没有想到一个好的原因，我认为读写按照相同的顺序完全可以根据绝对地址来实现数据写入和定位读取，这个大家想到有什么好的原因可以来讨论一下。

### FlatBuffers寻址特点  
除了Struct类型和基本类型，FlatBuffers在ByteBuffer中存放的数据地址都是相对地址，也使用相对寻址的方式来在ByteBuffer中定位数据。存在这个特点的根本原因是因为FlatBuffers存放数据和解析数据的方向不一致造成的。因此在向ByteBuffer写入数据的时候先暂时使用*offset*来定义位置，*offset*实际上就是指定位置和ByteBuffer结尾（capacity）的距离，但是又因为之后解析数据的时候并不是从后向前解析的，因此在解析的时候不能依赖这个*offset*。那要通过何种手段来确定一个成员的位置呢?FlatBuffers采用的是相对位置。例如，一个复杂数据类型在Table数据字段中存储的是这个复杂数据真正存储位置和Table数据字段写入位置的相对位置，这样只要已知Table数据字段的写入位置，就能计算得到这个复杂数据的在ByteBuffer中的真正存储位置了。下面根据上面介绍过的实例并结合源码来以String类型在Table中的存储和写入来解释一下这个特点。  
首先我们使用下面的代码来存储一个字符串数据到ByteBuffer中

	  int monsterName = builder.createString("软泥麦塔");
  
下面我们具体分析一下这个函数到底做了什么工作，是如何将字符串存储起来的，列出FlatBuffersBuilder中几个相关函数：

	/**
     * Encode the string `s` in the buffer using UTF-8.
     * 创建字符串，从左到右依次是字符串长度，字符串数据和结尾的0
     * @param s The string to encode.
     * @return The offset in the buffer where the encoded string starts.
     */
    public int createString(String s) {
        byte[] utf8 = s.getBytes(utf8charset);
        addByte((byte) 0);                  // 以0结尾
        startVector(1, utf8.length, 1);     // 也是按照vector进行存储的,当做长度为utf8.length的vector
        bb.position(space -= utf8.length);  // 存储字符串，注意顺序
        bb.put(utf8, 0, utf8.length);       // 存储字符串
        return endVector();
    }
    
    /**
     * Add a `byte` to the buffer, properly aligned, and grows the buffer (if necessary).
     *
     * @param x A `byte` to put into the buffer.
     */
    public void addByte(byte x) {
        prep(1, 0);
        putByte(x);
    }
    
    /**
     * Add a `byte` to the buffer, backwards from the current location. Doesn't align nor
     * check for space.
     *
     * @param x A `byte` to put into the buffer.
     */
    public void putByte(byte x) {
        bb.put(space -= 1, x);
    }
    
    public void startVector(int elem_size, int num_elems, int alignment) {
        notNested();
        vector_num_elems = num_elems;
        prep(SIZEOF_INT, elem_size * num_elems);    // 初始化存储空间
        prep(alignment, elem_size * num_elems); // Just in case alignment > int.
        nested = true;
    }
    
    /**
     * Finish off the creation of an array and all its elements.  The array
     * must be created with {@link #startVector(int, int, int)}.<br>
     * 结束vector的创建，同时会将此vector的长度信息写入，同时返回当前vector相对于ByteBuffer的偏移
     * @return The offset at which the newly created array starts.<br>
     *     当前创建的vector相对于ByteBuffer结尾的偏移
     * @see #startVector(int, int, int)
     */
    public int endVector() {
        if (!nested)
            throw new AssertionError("FlatBuffers: endVector called without startVector");
        nested = false;
        putInt(vector_num_elems);   // 保存vector的成员个数，不是存储空间长度
        return offset();    // 当前bytebuffer存放数据的长度
    }
    
    /**
     * Offset relative to the end of the buffer.<br>
     * 能够反映当前bytebuffer中存储数据的长度，也能表明当前对象相对于bytebuffer结尾的偏移
     * @return Offset relative to the end of the buffer.
     */
    public int offset() {
        return bb.capacity() - space;
    }
    
createString函数首先将字符串按照utf-8的方式进行了编码，并且在存储字符串数据之前先写了一个字节的0，以此作为字符串存储结尾的标志。大家注意下putByte这个函数的实现，可以发现每当FlatBuffers向ByteBuffer中写入数据的时候，都是先将*space*往ByteBuffer的头部移动指定长度，然后再写入数据，*space*的初始值为ByteBuffer.capacity，它维护了FlatBuffers向当前ByteBuffer写入位置信息，作用就类似于ByteBuffer原生的postion，是FlatBuffers逆向写入数据的产物。  
FlatBuffers在实现字符串写入的时候将字符串的编码数组当做了一维的vector来实现，startVector函数是写入前的初始化，并且在写入编码数组之前我们又看到了先将space往前移动数组长度的距离，然后再写入，写入完成后调用endVector进行收尾，endVector再将vector的成员数量，在这里就是字符串数组的长度写入，然后调用offset返回写入的数据结构的起点。在进行下一步分析之前，可以根据上面的分析画出写入数据的结构，如下：

![02_FlatBuffers中写入string的结构](/assets/posts/2016-03-14-android-flatbuffers/02_FlatBuffers中写入string的结构.png)

之后就是将字符串写入Table，下面列出相关代码：

	/**
	 * Monster经过flatc编译后自动生成的代码
	 */
	public static void addName(FlatBufferBuilder builder, int nameOffset) {
        builder.addOffset(3, nameOffset, 0);
    }

	/**
     * Adds on offset, relative to where it will be written.
     * 计算并保存指定off偏移与当前table_data第i个成员数据字段之间的偏移，
     * 方便以后找到第i个成员的数据位置后能够根据这个值计算出真正第i个数据的存储位置
     * @param off The offset to add.<br>相对于整个ByteBuffer的偏移
     */
    public void addOffset(int off) {
        prep(SIZEOF_INT, 0);  // Ensure alignment is already done.
        assert off <= offset();
        off = offset() - off + SIZEOF_INT; // 计算相对于当前写入位置的偏移
        putInt(off);
    }

实际上Table的数据结构比较复杂，后面会单独分析，这里只讲string在Table中的存储的关键代码。当调用addName将字符串存储到ByteBuffer的时候，需要传入刚才调用*builder.createString*函数返回的字符串的offset，然后addOffset会计算出传入的offset相对于当前写入位置的偏移，并将这个偏移写入。这意味着什么？意味着只要定位到当前写入的这个位置，取出写入的int值，和当前位置相加就得到了存储string数据的真实地址。这也是后面取string数据的一个依据，下面就来分析。


	/**
	 * Monster经过flatc编译后自动生成的代码
	 */
	public String name() {
        int o = __offset(10);	  // 得到vtable[10]相对于vatble_data[10]的偏移
        return o != 0 ? __string(o + bb_pos) : null;
    }
    
    
    /**
     * Create a Java `String` from UTF-8 data stored inside the FlatBuffer.
     * <p/>
     * This allocates a new string and converts to wide chars upon each access,
     * which is not very efficient. Instead, each FlatBuffer string also comes with an
     * accessor based on __vector_as_bytebuffer below, which is much more efficient,
     * assuming your Java program can handle UTF-8 data directly.
     *
     * @param offset An `int` index into the Table's ByteBuffer.<br>
     * @return Returns a `String` from the data stored inside the FlatBuffer at `offset`.
     */
    protected String __string(int offset) {
        offset += bb.getInt(offset);  // 当前位置和当前位置写入的相对偏移相加得到真实数据的位置
        if (bb.hasArray()) {    // ByteBuffer底层是否由byte数组支持
            return new String(bb.array(), bb.arrayOffset() + offset + SIZEOF_INT, bb.getInt(offset),
                    FlatBufferBuilder.utf8charset);
        } else {
            // We can't access .array(), since the ByteBuffer is read-only,
            // off-heap or a memory map
            ByteBuffer bb = this.bb.duplicate().order(ByteOrder.LITTLE_ENDIAN);
            // We're forced to make an extra copy:
            byte[] copy = new byte[bb.getInt(offset)];  // string第一个int表示字符串数组的长度，因此bb.getint(offset)就是string有效数据的长度
            bb.position(offset + SIZEOF_INT); // 要多偏移一个整型长度 ，以便跳过记录字符串的长度的数据，这样才能拷贝到真正的有效数据
            bb.get(copy);
            return new String(copy, 0, copy.length, FlatBufferBuilder.utf8charset); // 根据二进制构造字符串
        }
    }

这里先不讲*name()*函数如何通过vtable定位到table的数据字段，先认为*__string()* 函数传入的offset参数即是刚才写入string相对偏移的地址即可，从*__string()*函数的实现可以看到，首先是通过当前位置和写入的偏移计算出string数据存储的真正位置，然后根据string数据的存储格式取到string的真正数据，结束。  
从上面分析流程可以看出，在FlatBuffers对ByteBuffer写入顺序和读取顺序不一致的情况，使用相对寻址都不用关心我们当前读取和写入的顺序这个细节。

### FlatBuffers部分复杂数据结构存储分析 ###
有了FlatBuffers数据存储结构的基础后，就可以紧接着分析FlatBuffers支持的几个复杂数据结构的存储了。

#### Struct类型 ####
除了基本类型之外，FlatBuffers中只有Struct类型使用直接寻址进行数据访问。因此首先就来分析这种数据结构是如何进行存储的，还是结合之前的demo进行讲解。首先来看看Struct结构的java类实现：

	/**
	 * All structs in the generated code derive from this class, and add their own accessors.
	 */
	public class Struct {
	  /** Used to hold the position of the `bb` buffer. */
	  protected int bb_pos;
	  /** The underlying ByteBuffer to hold the data of the Struct. */
	  protected ByteBuffer bb;
	}
非常简单，只有两个成员变量，*bb*就是FlatBuffers用来存储数据的ByteBuffer，*bb_post*用来指明这个Struct对应的真实数在ByteBuffer中的绝对位置。这两个成员只描述了Struct最基本需求，至于Struct有几个成员变量，以及如何从底层ByteBuffer中解析Struct中的各个对象则留给了Struct的子类来实现。例如，我们可以看看Vec3的实现：

	public final class Vec3 extends Struct {
	    public Vec3 __init(int _i, ByteBuffer _bb) {
	        bb_pos = _i;
	        bb = _bb;
	        return this;
	    }
	
	    public float x() {
	        return bb.getFloat(bb_pos + 0);
	    }
	
	    public float y() {
	        return bb.getFloat(bb_pos + 4);
	    }
	
	    public float z() {
	        return bb.getFloat(bb_pos + 8);
	    }
	
	    public static int createVec3(FlatBufferBuilder builder, float x, float y, float z) {
	        builder.prep(4, 12);  // 准备，进行对齐及ByteBuffer长度不够时进行自动增长
	        builder.putFloat(z);
	        builder.putFloat(y);
	        builder.putFloat(x);
	        return builder.offset();
	    }
	};

从这个实现可以看到，子类自己维护了成员写入和解析的工作，例如自动根据索引写入x,y,z数据，以及根据索引从ByteBuffer中解析对应的成员变量等。因此，只要保证调用*Vec3.__init*方法的时候，*i*参数和调用*Vec3.createVec3*时返回的偏移相同，就一定可以成功从ByteBuffer中解析出数据。

#### Union类型 ####
这个类型也比较特殊，FlatBuffers规定这个类型在使用上具有如下两个限制：

1. Union类型的成员只能是Table类型。
2. Union类型不能是一个schema文件的根。

实际上BF中没有任何类型来表示Union，在schema中指明类型为Union后经过flatc编译后会生成一个单独的类来对应声明的Union类型。例如demo中的Equipment就为Union类型，声明为：

	union Equipment { Weapon }
	
经过编译后，生成的Equipment类如下：

	public final class Equipment {
	  private Equipment() { }
	  public static final byte NONE = 0;
	  public static final byte Weapon = 1;
	
	  private static final String[] names = { "NONE", "Weapon", };
	
	  public static String name(int e) { return names[e]; }
	};	

Union类的实现只是保存了Union类能够存储和表示的类型的名称，为了能够实现类似于联合体的功能，在编译Union类型的时候会为使用这个Union类型的外部类型额外生成一个代表当前Union类型的type，这个type和生成的Union类中的某一个常量type对应。真是因为需要生成一个额外的type类型和Union对应，FlatBuffers才限制了Union类型不能作为schema的根。例如，在demo中的Monster类中有如下代码就是单独为其中的Equipment生成的：

	public byte equippedType() {
        int o = __offset(20);
        return o != 0 ? bb.get(o + bb_pos) : 0;
    }

    public Table equipped(Table obj) {
        int o = __offset(22);
        return o != 0 ? __union(obj, o) : null;
    }
	
	public static void addEquippedType(FlatBufferBuilder builder, byte equippedType) {
        builder.addByte(8, equippedType, 0);
    }

    public static void addEquipped(FlatBufferBuilder builder, int equippedOffset) {
        builder.addOffset(9, equippedOffset, 0);
    }
    
因此，在序列化Union的时候一般先写入Union的type，然后再写入Union的数据偏移；在反序列化Union的时候一般先解析出Union的type，然后再按照type对应的Table类型来解析Union对应的数据。

#### enum类型 ####
FlatBuffers中的enum类型在数据存储的时候是和byte类型存储的方式一样的，因为和Union类型相似，enum类型在FlatBuffers中也没有单独的类与它对应，在schema中声明为enum的类会被编译生成单独的类。例如demo中的Color类被编译转换成了如下代码：

	public final class Color {
	  private Color() { }
	  public static final byte Red = 0;
	  public static final byte Green = 1;
	  public static final byte Blue = 2;
	
	  private static final String[] names = { "Red", "Green", "Blue", };
	
	  public static String name(int e) { return names[e]; }
	};

从类的实现上来看，Color类型只是简单地包括了枚举成员的声明和获取枚举成员名称的接口实现，在序列化和反序列化时，完全是将enum类型当做byte来处理的。例如Monster中对Color的处理如下：

	public static void addColor(FlatBufferBuilder builder, byte color) {
        builder.addByte(6, color, 2);
    }
    
    public byte color() {
        int o = __offset(16);
        return o != 0 ? bb.get(o + bb_pos) : 2;  // 默认颜色为Blue，因此没有存储颜色的时候使用2
    }

可以看到Monster类对Color的序列化和反序列化完全是按照byte来处理的。

#### Vector类型
Vector类型实际上就是schema中声明的数组类型，FlatBuffers中也没有单独的类型和它对应，但是它却有自己独立的一套存储结构，flatc编译生成的类负责按照这种结构来读写自己所用到的Vector类型的成员。Vector类型的存储结构如下：

![03_flatbuffers中vector类型的存储结构](/assets/posts/2016-03-14-android-flatbuffers/03_flatbuffers中vector类型的存储结构.png)

Vector在序列化数据时先会从高位到低位依次存储vector内部的数据，然后再在数据序列化完毕后写入Vector的成员个数。下面我们根据flatc生成的相关代码来看FlatBuffers是如何实现这个存储结构的：
	
	/********** Monster.java **********/
	
	public static void addWeapons(FlatBufferBuilder builder, int weaponsOffset) {
        builder.addOffset(7, weaponsOffset, 0);
    }

    public static int createWeaponsVector(FlatBufferBuilder builder, int[] data) {
        builder.startVector(4, data.length, 4);
        for (int i = data.length - 1; i >= 0; i--) {
            /** 
             * 由于weapon属于复杂类型，因此这里用addOffset记录成员变量，如果只是简单类型的vector，
             * flatc处理的时候会自动调用addInt等方法来记录简单数据，而不用进行相对寻址了。
             */
            builder.addOffset(data[i]);     
        }
        return builder.endVector();
    }

    public static void startWeaponsVector(FlatBufferBuilder builder, int numElems) {
        builder.startVector(4, numElems, 4);
    }
    
    
    public Weapon weapons(int j) {
        return weapons(new Weapon(), j);
    }

    public Weapon weapons(Weapon obj, int j) {
        int o = __offset(18);
        // 由于weapon vector成员记录的是各个weapon数据的存储地址相对成员数据写入位置的相对偏移，因此还要通过__indirect进行一次相对寻址
        return o != 0 ? obj.__init(__indirect(__vector(o) + j * 4), bb) : null;
    }

    public int weaponsLength() {
        int o = __offset(18);
        return o != 0 ? __vector_len(o) : 0;
    }
    
    /*********** Table.java ***********/

    /**
     * Look up a field in the vtable.<br>
     * 得到vtable中记录的第vtable_index个成员的值（是相应成员在数据字段中的偏移）
     * @param vtable_index An `int` offset to the vtable in the Table's ByteBuffer.
     * @return Returns an offset into the object, or `0` if the field is not present.
     *          得到vtable_offset指定的table成员相对table data中的位置
     */
    protected int __offset(int vtable_index) {
        int vtable = bb_pos - bb.getInt(bb_pos);    // bb_pos得到的是table的data字段相对于vtable的偏移，因此这种方法得到的是vatable的位置
        return vtable_index < bb.getShort(vtable) ? bb.getShort(vtable + vtable_index) : 0;   // 因为vtable_offset是相对于vtable的偏移，因此vtable_offset < bb.getShort(vtable)可以防止偏移越界
    }

    /**
     * Retrieve a relative offset.<br>
     * 进行一次相对寻址
     * @param offset An `int` index into the Table's ByteBuffer containing the relative offset.
     * @return Returns the relative offset stored at `offset`.
     */
    protected int __indirect(int offset) {
        return offset + bb.getInt(offset);
    }
    
    /**
     * Get the length of a vector.
     *
     * @param offset An `int` index into the Table's ByteBuffer.
     * @return Returns the length of the vector whose offset is stored at `offset`.
     */
    protected int __vector_len(int offset) {
        offset += bb_pos;  // 加上byteBuffer数据段在Table中的偏移。
        offset += bb.getInt(offset);  // ByteBuffer中数据段的偏移
        return bb.getInt(offset); // 得到数据段中表明有效数据长度的值
    }

    /**
     * Get the start data of a vector.<br>
     * 得到vector中有效数据相对于整个ByteBuffer的偏移
     *
     * @param offset An `int` index into the Table's ByteBuffer.
     * @return Returns the start of the vector data whose offset is stored at `offset`.
     */
    protected int __vector(int offset) {
        offset += bb_pos;   // 得到Table中第offset个成员的地址
        return offset + bb.getInt(offset) + SIZEOF_INT;  // data starts after the length，相对寻址，得到vector数据的真正地址
    }
    
	
	/*************** FlatBuilder.java ***********/
	
	public void startVector(int elem_size, int num_elems, int alignment) {
        notNested();
        vector_num_elems = num_elems;
        prep(SIZEOF_INT, elem_size * num_elems);    // 初始化存储空间
        prep(alignment, elem_size * num_elems); // Just in case alignment > int.
        nested = true;
    }
	
    /**
     * Finish off the creation of an array and all its elements.  The array
     * must be created with {@link #startVector(int, int, int)}.<br>
     * 结束vector的创建，同时会将此vector的长度信息写入，同时返回当前vector相对于ByteBuffer的偏移
     * @return The offset at which the newly created array starts.<br>
     *     当前创建的vector相对于ByteBuffer结尾的偏移
     * @see #startVector(int, int, int)
     */
    public int endVector() {
        if (!nested)
            throw new AssertionError("FlatBuffers: endVector called without startVector");
        nested = false;
        putInt(vector_num_elems);   // 保存vector的成员个数，不是存储空间长度，除非vector是以byte为单位保存
        return offset();    // 当前bytebuffer存放数据的长度
    }

这里进行简单分析。首先从序列化数据开始，首先调用startVector进行对齐等初始化工作，然后依次写入Vector的成员变量。注意由于Vector的成员是复杂的Table类型，因此flatc在处理的时候自动使用了*addOffset*的方法来写入成员的相对偏移，这意味着后面要反序列化数据的时候需要取出这个偏移再进行一次相对寻址才能访问到复杂类型成员的真正数据；但是如果Vector的成员是简单类型，例如byte或者int时，flatc会自动调用*addByte*或者*addInt*等函数来直接存储成员数据，这样在反序列化时可以直接取出存储的数据就能代表成员变量的值。写入成员数据后再调用*endVector*将Vector的成员个数写入，就完成了序列化的工作。  
在反序列化的时候，先通过*__offset*得到Vector相对于外部Table数据字段的偏移，然后调用 *__vector*函数得到这个Vector真正存储数据的位置，但是刚才已经说明，由于Vector中存储的只是Weapon的相对地址，因此绝对偏移地址：__vector(o) + j x 4  写入的内容就是第j个Vector变量相对于写入地址的偏移，因此还要通过调用一次__indirect方法进行相对寻址才能得到Vector第j个成员Weapon的偏移地址。


#### Table类型 ####
Table类型是FlatBuffers中的核心类型，也是其中存储最为复杂的类型，首先我们来看Table类型的存储结构，如下图：	

![04_flatbuffers中Table存储结构](/assets/posts/2016-03-14-android-flatbuffers/04_flatbuffers中Table存储结构.png)

单就结构来讲就看出这种类型的复杂性了，不过有了之前讲解的几种结构的基础，一点点剖析起来也不难。  
首先可以将Table分为两个部分，第一部分是存储Table中各个成员变量的概要，这里命名为vtable，第二部分是Table的数据部分，存储Table中各个成员的值，这里命名为table_data。注意Table中的成员如果是简单类型或者Struct类型，那么这个成员的具体数值就直接存储在table_data中；如果成员是复杂类型，那么table_data中存储的只是这个成员数据相对于写入地址的偏移，也就是说要获得这个成员的真正数据还要取出table_data中的数据进行一次相对寻址，这个特点在上面已经强调过了，分析FlatBuffers的时候一定要牢记这个规则。  
下面就开始结合demo和源码来分析Table的存储及序列化和反序列化的操作过程。首先看一下demo中调用*startMonster*开始进行序列化的时候做了什么处理，相关代码如下：

	/********** Monster.java *************/
	
	public static void startMonster(FlatBufferBuilder builder) {
        builder.startObject(10);
    }
    
    
    /********* FlatBuilders.java ********/
    
    public void startObject(int numfields) {
        notNested();
        if (vtable == null || vtable.length < numfields)
            vtable = new int[numfields];
        vtable_in_use = numfields;
        Arrays.fill(vtable, 0, vtable_in_use, 0);
        nested = true;
        object_start = offset();
    }
    
做的处理主要是在内存中为当前Table分配一个vtable并初始化，vtable的长度和Table的成员变量数相同，实际上vtable中的每一项记录的就是Table中对应的成员在table_data中的偏移，这一点我们稍后会分析到。还要注意的是，这里仅仅是在内存中分配了一个vtable数组，用于暂时记录Table数据，而没有直接将这些数据写入到底层的ByteBuffer。同时，使用*object_start*来记录了Table数据开始的偏移。  
在调用了*startMonster*方法后就可以开始一步步添加Monster中的各个成员，Monster为添加各种成员封装了很多add方法，但是这些add方法对应到FlatBuffers的底层无非就两类：

* 基本类型  
基本类型的add方法包括addInt、addDouble、addBoolean等等，它们只是在ByteBuffer中说占用的存储长度有所不同，本质上都是将成员的数据直接进行存储。
* 偏移类型  
对应为addOffset方法，这个方法接受一个相对于底层ByteBuffer的绝对偏移，然后将这个偏移根据当前写入位置相对于底层ByteBuffer的绝对偏移计算得到相对偏移，然后再将这个相对偏移写入到底层ByteBuffer中。偏移类型在ByteBuffer中占4个字节，而且在底层的存储上和addInt没有任何差别，但是毕竟addOffset和addInt所表示的意思完全不同，因此FlatBuffers就通过flatc对schema的编译来保证生成的类一定遵循*"复杂类型的偏移用addOffset，简单类型使用addInt"*这种规则，并且*凡是使用addOffset进行序列化存储的数据在反序列化时一定会进行一次相对寻址*。  

这里可以从demo中择抄几段代码来进一步验证上面的分析。

	/********* Monster.java ********/
	public static void addMana(FlatBufferBuilder builder, short mana) {
        builder.addShort(1, mana, 150);
    }
    
    public short mana() {
        int o = __offset(6);
        return o != 0 ? bb.getShort(o + bb_pos) : 150;
    }
	
	public static void addName(FlatBufferBuilder builder, int nameOffset) {
        builder.addOffset(3, nameOffset, 0);
    }
    
    public String name() {
        int o = __offset(10);
        return o != 0 ? __string(o + bb_pos) : null;
    }
    
    /******** FlatBuilder.java *********/
    
    /**
     * Add a `short` to the buffer, properly aligned, and grows the buffer (if necessary).
     *
     * @param x A `short` to put into the buffer.
     */
    public void addShort(short x) {
        prep(2, 0);
        putShort(x);
    }
    
    /**
     * Add a `short` to the buffer, backwards from the current location. Doesn't align nor
     * check for space.
     *
     * @param x A `short` to put into the buffer.
     */
    public void putShort(short x) {
        bb.putShort(space -= 2, x);
    }
    
	/**
     * Adds on offset, relative to where it will be written.<br>
     * 写入相对偏移信息，相对的是当前写入位置的偏移。
     * @param off The offset to add.<br>相对于整个ByteBuffer的偏移
     */
    public void addOffset(int off) {
        prep(SIZEOF_INT, 0);  // Ensure alignment is already done.
        assert off <= offset();
        off = offset() - off + SIZEOF_INT;  // 相对寻址，得到的是相对于当前写入位置的偏移
        putInt(off);
    }
	
	/**
     * Add an `int` to the buffer, backwards from the current location. Doesn't align nor
     * check for space.
     *
     * @param x An `int` to put into the buffer.
     */
    public void putInt(int x) {
        bb.putInt(space -= 4, x);
    }
	
	protected String __string(int offset) {
        offset += bb.getInt(offset);  // 相对寻址
        if (bb.hasArray()) {    // ByteBuffer底层是否由byte数组支持
            return new String(bb.array(), bb.arrayOffset() + offset + SIZEOF_INT, bb.getInt(offset),
                    FlatBufferBuilder.utf8charset);
        } else {
            // We can't access .array(), since the ByteBuffer is read-only,
            // off-heap or a memory map
            ByteBuffer bb = this.bb.duplicate().order(ByteOrder.LITTLE_ENDIAN);
            // We're forced to make an extra copy:
            byte[] copy = new byte[bb.getInt(offset)];  // offset加上了偏移，因此bb.getint(offset)就是bytebuffer有效数据的长度
            bb.position(offset + SIZEOF_INT); // 要多加一个代表有效数据长度的偏移，这样才能拷贝到真正的有效数据
            bb.get(copy);
            return new String(copy, 0, copy.length, FlatBufferBuilder.utf8charset);
        }
    }

从以上代码可以看到，FlatBuilder虽然为不同类型的存储封装了不同的方法，但是在底层序列化存储数据的时候不区分要存储的数据是那一类类型，不同数据类型在序列化和反序列化时的正确对应是由flatc生成的类来保证的。  
当所有的数据存储完毕后，就可以调用*Monster.endMonster*方法来标志Monster数据存储的结束，这个函数一系列的操作很关键，下面来分析：

	/******** Monster.java ********/
	
	public static int endMonster(FlatBufferBuilder builder) {
        int o = builder.endObject();
        return o;
    }
	
	/**
     * Finish off writing the object that is under construction.
     *
     * @return The offset to the object inside {@link #dataBuffer()}.
     * @see #startObject(int)
     */
    public int endObject() {
        if (vtable == null || !nested)
            throw new AssertionError("FlatBuffers: endObject called without startObject");

        addInt(0);  // 中间间隔4个字节 用来记录vatable相对于table_data的结尾位置的偏移   ---> tag 2

        //----------- 记录vatable信息到bytebuffer START ------------

        int vtableloc = offset();
        // Write out the current vtable.
        for (int i = vtable_in_use - 1; i >= 0; i--) {  // 从后向前依次记录成员vtable的成员偏移到bytebuffer中
            // Offset relative to the start of the table.
            /**
             * 注意这里的相对偏移的计算：
             * vtableloc = capacity - vtablePos;   // table_data相对于bytebuffer结尾的偏移
             * vtable[i] = capacity -  tableDataPos_i;   // table数据段中第i个字段相对于bytebuffer结尾的偏移
             * 那么第i个字段相对于table_data的偏移计算为：
             * offset = vtableloc - vtable[i] = tableDataPos_i - vtablePos;
             *  那么在之后知道table_data相对于bytebuffer偏移的情况下，要得到这个成员记录的偏移就直接可以用：
             *
             * vtablePos + offset = tableDataPos_i;
             *
             * 参见 ：{@link Table#__offset(int)}
             */

            short off = (short) (vtable[i] != 0 ? vtableloc - vtable[i] : 0);   // 得到成员数据在table_data中的写入位置相对于table_data的偏移

            addShort(off);
        }

        final int standard_fields = 2; // The fields below: 因为后面还要写两个字段，因此这里是2
        addShort((short) (vtableloc - object_start));   // 再写入整个 table_data的长度
        addShort((short) ((vtable_in_use + standard_fields) * SIZEOF_SHORT));   // vtable 的长度   ---> tag 1

        //----------- 记录vatable信息到bytebuffer END   ------------

        // Search for an existing vtable that matches the current one.
        int existing_vtable = 0;
        outer_loop:
        for (int i = 0; i < num_vtables; i++) {
            int vt1 = bb.capacity() - vtables[i];       // vtables记录的是每个vtable的偏移
            int vt2 = space;    // 因为刚才才存储了一个vtable数据到bytebuffer，因此这里space就指向当前的vatable
            short len = bb.getShort(vt1);   // vtable的第一个short字段记录的就是该vatable的长度（见上面注释的tag 1）
            if (len == bb.getShort(vt2)) {
                for (int j = SIZEOF_SHORT; j < len; j += SIZEOF_SHORT) {
                    if (bb.getShort(vt1 + j) != bb.getShort(vt2 + j)) {
                        continue outer_loop;
                    }
                }
                existing_vtable = vtables[i];
                break outer_loop;
            }
        }

        if (existing_vtable != 0) {     // 存在和当前vtable一样的vatable
            // Found a match:
            // Remove the current vtable.
            space = bb.capacity() - vtableloc;  // 移动space到table_data的结尾
            // Point table to existing vtable.
            bb.putInt(space, existing_vtable - vtableloc);  // 记录当前table_data对应的vtable的位置（见 tag 2注释)
        } else {
            // No match:
            // Add the location of the current vtable to the list of vtables.
            if (num_vtables == vtables.length)
                vtables = Arrays.copyOf(vtables, num_vtables * 2);  // 扩容
            vtables[num_vtables++] = offset();  // 记录当前vtable
            // Point table to current vtable.
            bb.putInt(bb.capacity() - vtableloc, offset() - vtableloc); // 记录vatable相对于table的偏移位置
        }

        nested = false;
        return vtableloc;  // 注意返回的是table_data的偏移，不是vtable的偏移
    }

代码中的一堆注释已经解释了操作的各个步骤完成的任务，总的说来，就是完成了table_data的构建，同时将内存中的vtable写入到了底层的ByteBuffer中。但是有一点需要注意的是，一般来说vtable和table_data就和上面图中画出的一样，是相邻的，但是从代码中来看，如果在调用endObject的时候ByteBuffer中已经存在一个和当前table_data需要的vtable一样的vtable存在时，当前的table_data是会直接复用ByteBuffer中的这个vtable，因此就可能出现二者不相邻的情况，这也说明了:***一个vtable在ByteBuffer中可以对应多个table_data，因为vtable只记录Table结构相关的概要信息，和Table具体存储的数据无关***	。这也是FlatBuffers将Table数据拆成两部分的一个原因吧，因为如果ByteBuffer中有多个类型相同的Table数据时，这样可以节省存储空间。
	
![05_vtable复用](/assets/posts/2016-03-14-android-flatbuffers/05_vtable复用.png)

调用*endObject()*后就表示一个Table数据写入完毕，但是如果这个Table是整个schema的root_table，那么还需要调用*FlatBuilder.finish()*方法来在底层的ByteBuffer的开头写入这个Table的偏移，这样就能够在反序列化的时候通过读取ByteBuffer的前4个字节就能够确定root_table的位置并顺次解析数据。调用完*FlatBuilder.finish()*方法后就不能再往ByteBuffer中添加任何数据。

	/**
     * Finalize a buffer, pointing to the given `root_table`.
     *
     * @param root_table An offset to be added to the buffer.
     */
    public void finish(int root_table) {
        prep(minalign, SIZEOF_INT);
        addOffset(root_table);  // 写入root_table的table_data的偏移
        bb.position(space);     // 修改bytebuffer的指针位置，以指明ByteBuffer中的有效数据的开端
        finished = true;
    }

在解析Table数据的时候，FlatBuffers都是通过偏移先找到table_data的位置，然后再根据table_data开始的4个直接找到vtable，然后根据vtalbe来辅助table_data中的数据解析。例如：

	public static Monster getRootAsMonster(ByteBuffer _bb) {
        return getRootAsMonster(_bb, new Monster());
    }

    public static Monster getRootAsMonster(ByteBuffer _bb, Monster obj) {
        _bb.order(ByteOrder.LITTLE_ENDIAN);
        return (obj.__init(_bb.getInt(_bb.position()) + _bb.position(), _bb));  // _bb.getInt(_bb.position()) + _bb.position()得到的就是table_data在bytebuffer中的position
    }

    public Monster __init(int _i, ByteBuffer _bb) {
        bb_pos = _i;
        bb = _bb;
        return this;
    }
请注意，上面的代码中*_bb.getInt(_bb.position())+_bb.position()*得到的就是table_data的偏移，因为*FlatBuilder.endObject()*返回的是table_data的偏移，*FlatBuilder.finish()*又将这个偏移计算成了相对偏移并记录。因此Table中的bb_pos就是这个Table中的table_data字段在ByteBuffer中的偏移地址。
到这里基本上就将FlatBuffers中的Table讲解完毕了，只剩下最后一个小的细节，那就是：*对于简单数据，如果指定写入的数据和默认值相同，FlatBuffers是不会将此数据写入到底层ByteBuffer中的*，这又是FlatBuffers节省数据长度的一个优化。大家可以从下面的代码看到这个特点：

	/******* Monster.java *********/
	
	public static void addHp(FlatBufferBuilder builder, short hp) {
        builder.addShort(2, hp, 100);
    }
    
    public short hp() {
        int o = __offset(8);
        return o != 0 ? bb.getShort(o + bb_pos) : 100;
    }
    
    /******** FlatBuilder.java ********/
    /**
     * Add a `short` to a table at `o` into its vtable, with value `x` and default `d`.
     *
     * @param o The index into the vtable.
     * @param x A `short` to put into the buffer, depending on how defaults are handled. If
     *          `force_defaults` is `false`, compare `x` against the default value `d`. If `x` contains the
     *          default value, it can be skipped.
     * @param d A `short` default value to compare against when `force_defaults` is `false`.
     */
    public void addShort(int o, short x, int d) {
        if (force_defaults || x != d) {
            addShort(x);
            slot(o);
        }
    }
    
    /**
     * Set the current vtable at `voffset` to the current location in the buffer.<br>
     * 使vtable的第vtableIndex记录当前bytebuffer的偏移。一般用于vtable记录table中字段的位置
     *
     * @param vtableIndex The index into the vtable to store the offset relative to the end of the
     *                buffer.<br>
     *                table中成员的索引
     */
    public void slot(int vtableIndex) {
        vtable[vtableIndex] = offset();
    }
    
以上代码*FlatBuilder.addShort()*方法在force_defaults为false的情况下，如果写入的值和当前值相同，那么并不会将数据写入到table_data中，相应的*vtable[i]*就为0，并且后面通过*FlatBuilder.endObject()*写入到ByteBuffer中的vtable的第i个成员也为0。此后，调用*Monster.hp()*的时候*__offset()*函数返回的值就是0，在这种情况下，FlatBuffers不会再去table_data中去寻找成员的值或者偏移，而是直接返回了schema中规定的默认值。同理，复杂类型数据也有和简单数据类型类似的处理。

## 结束 ##
本文结合demo和源码对FlatBuffers进行了剖析，在解释原理的时候为了方便省略了其中的一些细节，对于省略的内容并不是不重要，比如说其中的对齐操作和相对偏移的计算都是至关重要的，大家可以参考本文大概把握FlatBuffers的原理，然后对其中的一些细节使用自己的方法来理解。本文的内容是我个人总结，如有偏颇，烦请指正！

## 参考 ##
[FlatBuffer Overview](http://google.github.io/flatbuffers/index.html#flatbuffers_overview)
    
    


