---
layout: post
title: protobuf语言指南
category: 工具
description:
---
**定义消息类型**  

	message SearchRequest {
  		required string query = 1;
		optional int32 page_number = 2;
		optional int32 result_per_page = 3;
	}  

- 指定字段类型  
在上面的例子中，所有字段都是标量类型：两个整型（page_number和result_per_page），一个string类型（query）。当然，你也可以为字段指定其他的合成类型，包括枚举（enumerations）或其他消息类型。  
- 分配标识号  
正如上述文件格式，在消息定义中，每个字段都有唯一的一个标识符。这些标识符是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改变。注：[1,15]之内的标识号在编码的时候会占用一个字节。[16,2047]之内的标识号则占用2个字节。所以应该为那些频繁出现的消息元素保留 [1,15]之内的标识号。  
- 指定字段规则  
所指定的消息字段修饰符必须是如下之一：  
- required：一个格式良好的消息一定要含有1个这种字段。表示该值是必须要设置的；
- optional：消息格式中该字段可以有0个或1个值（不超过1个）。
- repeated：在一个格式良好的消息中，这种字段可以重复任意多次（包括0次）。重复的值的顺序会被保留。表示该值可以重复。  

如上所述，消息描述中的一个元素可以被标记为可选的”（optional）。一个格式良好的消息可以包含0个或一个optional的元素。当解析消息时，如果它不包含optional的元素值，那么解析出来的对象中的对应字段就被置为默认值。默认值可以在消息描述文件中指定:  

	optional int32 result_per_page = 3 [default = 10];  

在一个.proto文件中可以定义多个消息类型,向.proto文件添加注释，可以使用C/C++/java风格的双斜杠（//） 语法格式。对C++来说，编译器会为每个.proto文件生成一个.h文件和一个.cc文件，.proto文件中的每一个消息有一个对应的类。对Java来说，编译器为每一个消息类型生成了一个.java文件，以及一个特殊的Builder类（该类是用来创建消息类接口的）。对Python来说，有点不太一样——Python编译器为.proto文件中的每个消息类型生成一个含有静态描述符的模块，该模块与一个元类（metaclass）在运行时（runtime）被用来创建所需的Python数据访问类。  

**标量数值类型**  
<table class="table table-bordered table-striped table-condensed">
<tr>
<th width="100">proto类型</th>
<th width="100">python类型</th>
<th width="100">C++类型</h>
<th>备注</th>
</tr>
<tr>
<td>double</td>
<td>float</td>
<td>double</td>
</tr>
<tr>
<td>float</td>
<td>float</td>
<td>float</td>
</tr>
<tr>
<td>int32</td>
<td>int</td>
<td>int32</td>
<td>使用可变长编码方式。编码负数时不够高效——如果你的字段可能含有负数，那么请使用sint32。</td>
</tr>
<tr>
<td>int64</td>
<td>int/long</td>
<td>int64</td>
<td>使用可变长编码方式。编码负数时不够高效——如果你的字段可能含有负数，那么请使用sint64。</td>
</tr>
<tr>
<td>uint32</td>
<td>int/long</td>
<td>uint32</td>
<td>Uses variable-length encoding.</td>
</tr>
<tr>
<td>uint64</td>
<td>int/long</td>
<td>uint64</td>
<td>Uses variable-length encoding.</td>
</tr>
<tr>
<td>sint32</td>
<td>int</td>
<td>int32</td>
<td>使用可变长编码方式。有符号的整型值。编码时比通常的int32高效。</td>
</tr>
<tr>
<td>sint64</td>
<td>int/long</td>
<td>int64</td>
<td>使用可变长编码方式。有符号的整型值。编码时比通常的int64高效。</td>
</tr>
<tr>
<td>fixed32</td>
<td>int</td>
<td>uint32</td>
<td>总是4个字节。如果数值总是比总是比228大的话，这个类型会比uint32高效。</td>
</tr>
<tr>
<td>fixed64</td>
<td>int/long</td>
<td>uint64</td>
<td>总是8个字节。如果数值总是比总是比256大的话，这个类型会比uint64高效。</td>
</tr>
<tr>
<td>sfixed32</td>
<td>int</td>
<td>int32</td>
<td>总是4个字节。</td>
</tr>
<tr>
<td>sfixed64</td>
<td>int/long</td>
<td>int64</td>
<td>总是8个字节。</td>
</tr>
<tr>
<td>bool</td>
<td>boolean</td>
<td>bool</td>
</tr>
<tr>
<td>string</td>
<td>str/unicode</td>
<td>string</td>
<td>一个字符串必须是UTF-8编码或者7-bit ASCII编码的文本。</td>
</tr>
<tr>
<td>bytes</td>
<td>str</td>
<td>string</td>
<td>可能包含任意顺序的字节数据。</td>
</tr>
</table>  

**枚举**  

当需要定义一个消息类型的时候，可能想为一个字段指定某“预定义值序列”中的一个值。枚举常量必须在32位整型值的范围内。因为enum值是使用可变编码方式的，对负数不够高效，因此不推荐在enum中使用负数。当对一个使用了枚举的.proto文件运行protocol buffer编译器的时候，生成的代码中将有一个对应的enum（对Java或C++来说），或者一个特殊的EnumDescriptor类（对Python来说），它被用来在运行时生成的类中创建一系列的整型值符号常量（symbolic constants）。  

**使用其他消息类型**  

可以将其他消息类型用作字段类型。例如，假设在每一个SearchResponse消息中包含Result消息，此时可以在相同的.proto文件中定义一个Result消息类型，然后在SearchResponse消息中指定一个Result类型的字段，如：  

	message SearchResponse {
	repeated Result result = 1;
	}
	message Result {
		required string url = 1;
  		optional string title = 2;
  		repeated string snippets = 3;
	}  

在上面的例子中，Result消息类型与SearchResponse是定义在同一文件中的。如果想要使用的消息类型已经在其他.proto文件中已经定义过了呢？你可以通过导入（importing）其他.proto文件中的定义来使用它们。要导入其他.proto文件的定义，你需要在你的文件中添加一个导入声明，如：  

	import "myproject/other_protos.proto";  

protocol编译器就会在一系列目录中查找需要被导入的文件，这些目录通过protocol编译器的命令行参数-I/–import_path指定。如果不提供参数，编译器就在其调用目录下查找。  

**嵌套类型**  

你可以在其他消息类型中定义、使用消息类型，在下面的例子中，Result消息就定义在SearchResponse消息内，如：  

	message SearchResponse {
  		message Result {
    	required string url = 1;
    	optional string title = 2;
    	repeated string snippets = 3;
  		}
 		repeated Result result = 1;
	}  

如果你想在它的父消息类型的外部重用这个消息类型，你需要以Parent.Type的形式使用它，如：  

	message SomeOtherMessage {
  		optional SearchResponse.Result result = 1;
	}  

**更新一个消息类型**  

如果一个已有的消息格式已无法满足新的需求——如，要在消息中添加一个额外的字段——但是同时旧版本写的代码仍然可用。不用担心！更新消息而不破坏已有代码是非常简单的。在更新时只要记住以下的规则即可。  


- 不要更改任何已有的字段的数值标识。
- 所添加的任何字段都必须是optional或repeated的。这就意味着任何使用“旧”的消息格式的代码序列化的消息可以被新的代码所解析，因为它们 不会丢掉任何required的元素。应该为这些元素设置合理的默认值，这样新的代码就能够正确地与老代码生成的消息交互了。类似地，新的代码创建的消息 也能被老的代码解析：老的二进制程序在解析的时候只是简单地将新字段忽略。然而，未知的字段是没有被抛弃的。此后，如果消息被序列化，未知的字段会随之一 起被序列化——所以，如果消息传到了新代码那里，则新的字段仍然可用。注意：对Python来说，对未知字段的保留策略是无效的。
- 非required的字段可以移除——只要它们的标识号在新的消息类型中不再使用（更好的做法可能是重命名那个字段，例如在字段前添加“OBSOLETE_”前缀，那样的话，使用的.proto文件的用户将来就不会无意中重新使用了那些不该使用的标识号）。
- 一个非required的字段可以转换为一个扩展，反之亦然——只要它的类型和标识号保持不变。
- int32, uint32, int64, uint64,和bool是全部兼容的，这意味着可以将这些类型中的一个转换为另外一个，而不会破坏向前、 向后的兼容性。如果解析出来的数字与对应的类型不相符，那么结果就像在C++中对它进行了强制类型转换一样（例如，如果把一个64位数字当作int32来 读取，那么它就会被截断为32位的数字）。
- sint32和sint64是互相兼容的，但是它们与其他整数类型不兼容。
- string和bytes是兼容的——只要bytes是有效的UTF-8编码。
- 嵌套消息与bytes是兼容的——只要bytes包含该消息的一个编码过的版本。
- fixed32与sfixed32是兼容的，fixed64与sfixed64是兼容的。  

**扩展**  

通过扩展，可以将一个范围内的字段标识号声明为可被第三方扩展所用。然后，其他人就可以在他们自己的.proto文件中为该消息类型声明新的字段，而不必去编辑原始文件了:  

	message Foo {
  		// …
  		extensions 100 to 199;
	}  

在消息Foo中，范围[100,199]之内的字段标识号被保留为扩展用。现在，其他人就可以在他们自己的.proto文件中添加新字段到Foo里了，但是添加的字段标识号要在指定的范围内:   

	extend Foo {
  		optional int32 bar = 126;
	}  

**包**  

可以为.proto文件新增一个可选的package声明符，用来防止不同的消息类型有命名冲突:  

	package foo.bar;
	message Open { ... }  

在其他的消息格式定义中可以使用包名+消息名的方式来定义域的类型:   

	message Foo {
		...
  		required foo.bar.Open open = 1;
 		...
	}  

**定义服务**  

如果想要将消息类型用在RPC(远程方法调用)系统中，可以在.proto文件中定义一个RPC服务接口，protocol buffer编译器将会根据所选择的不同语言生成服务接口代码及存根。如，想要定义一个RPC服务并具有一个方法，该方法能够接收 SearchRequest并返回一个SearchResponse，此时可以在.proto文件中进行如下定义：  

	service SearchService {
  		rpc Search (SearchRequest) returns (SearchResponse);
	}  

**选项**  

在定义.proto文件时能够标注一系列的options。Options并不改变整个文件声明的含义，但却能够影响特定环境下处理方式。完整的可用选项可以在google/protobuf/descriptor.proto找到。
一些选项是文件级别的，意味着它可以作用于最外范围，不包含在任何消息内部、enum或服务定义中。一些选项是消息级别的，意味着它可以用在消息定 义的内部。当然有些选项可以作用在域、enum类型、enum值、服务类型及服务方法中。到目前为止，并没有一种有效的选项能作用于所有的类型。  
如下就是一些常用的选择：  

- java_package (file option)
- java_outer_classname (file option)
- optimize_for (fileoption): 可以被设置为 SPEED, CODE_SIZE,or LITE_RUNTIME。
- cc_generic_services, java_generic_services, py_generic_services (file options)
- message_set_wire_format (message option)
- packed (field option): 如果该选项在一个整型基本类型上被设置为真，则采用更紧凑的编码方式。
- deprecated (field option): 如果该选项被设置为true，表明该字段已经被弃用了。  

ProtocolBuffers允许自定义并使用选项，由于options是定在 google/protobuf/descriptor.proto中的，因此你可以在该文件中进行扩展，定义自己的选项:  

	import "google/protobuf/descriptor.proto";
	extend google.protobuf.MessageOptions {
  		optional string my_option = 51234;
	}  
	message MyMessage {
  		option (my_option) = "Hello world!";
	}  

在上述代码中，通过对MessageOptions进行扩展定义了一个新的消息级别的选项。当使用该选项时，选项的名称需要使用（）包裹起来，以表明它是一个扩展。在C++代码中可以看出my_option是以如下方式被读取的:  

	string value = MyMessage::descriptor()->options().GetExtension(my_option);  

**生成代码**　　

可以通过定义好的.proto文件来生成Java、Python、C++代码，需要基于.proto文件运行protocolbuffer编译器protoc。运行的命令如下所示：  

	protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR path/to/file.proto  
    
 
- IMPORT_PATH声明了一个.proto文件所在的具体目录。如果忽略该值，则使用当前目录。如果有多个目录则可以 对--proto_path 写多次，它们将会顺序的被访问并执行导入。-I=IMPORT_PATH是它的简化形式。
- --cpp_out 在目标目录DST_DIR中产生C++代码。
- --java_out 在目标目录DST_DIR中产生Java代码。
- --python_out 在目标目录 DST_DIR 中产生Python代码。  



