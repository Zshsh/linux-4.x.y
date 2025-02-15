基于fuse文件系统优化方法总结

        目前很多文件系统基于Fuse（ http://fuse.sourceforge.net/ ）开发，作者深入钻研Fuse代码后，总结出开发此类文件系统时可考虑的优化方案，拿出来与大家讨论讨论，如有不准确的地方，还望大家不吝赐教。阅读本文前，我假设你对Fuse有了足够多的了解（起码知道Fuse有两个模块：Fuse Kernel 和LibFuse以及知道一个应用程序调用行为如何传递至我们自己开发的基于Fuse的文件系统），否则，请先移步。


　　? 优化1：延长元数据有效时间


　　Linux中每个打开文件在内核中拥有两种元数据信息：struct dentry和struct inode，它们是文件在内核的基础。所有对文件的操作，都需要先获取文件这两个结构方可继续下去，而这两个结构又是由具体文件系统负责构造填充。以下两点解释了元数据优化的必要性：


　　1）。 应用程序调用文件系统操作系统接口时，传入的参数一般为文件路径，如open（“a/b/c/d.txt”），内核需要对路径名进行解析，从根目录开始，根据路径中的每个分量获取其dentry和inode，接着 加粗文字 解析路径的下一个分量，直至解析出目的文件的inode和dentry，如果路径名分量中的dentry没有缓存在内存中，需要从具体文件系统上读出（这就耗时多了）。


　　2）。 很多应用程序喜欢调用stat接口以获取文件属性，内核实现其实是找到文件inode，从inode中获取文件属性。如果inode没有被缓存，则需要从具体文件系统中获取（可能会很耗时）。


　　因为Fuse的内核模块只是一个桥梁，连接了应用程序和我们基于Fuse开发的文件系统。所以，按照道理说，每次获取文件/目录的inode以及dentry的时候Fuse内核模块都应该去LibFuse以及我们的文件系统走一遭。


　　但是这样做的话缺点非常明显：IO路径拉长，效率变低，而且假如我们基于fuse开发的文件系统是网络文件系统（例如NOS等），可能会导致后端服务器压力增大。


　　有鉴于此，Fuse的作者在Kernel Fuse模块中增加了元数据缓存，包含dentry和inode缓存。相比本地文件系统，我们必须时刻警惕一个问题：缓存有效性。所以，如何在提升性能的同时又尽量保证正确性是一个棘手的问题。


　　利用fuse挂载我们自己文件系统时，可指定dentry以及inode属性有效时间，当然这个有效时间得具体问题具体设置了，无统一答案。


　　优化方法：fuse挂载指定 –o entry_timeout=T –o attr_timeout=T


　　优化建议：五颗星


　　? 优化2：扩大每次写入页面数


　　应用程序每次对基于Fuse开发的文件系统的文件写入必先经过Kernel Fuse模块，Kernel Fuse其实是有很大权限决定何时将数据写入到用户态文件系统的。写的越频繁，效率必然越低，但一致性可能会更好，控制写入频率其实也是一个权衡的过程。


　　如果稍微熟悉Kernel你可能就会知道内核的IO其实是以Page为单位的。内核会将应用程序的写入请求按照PAGE_SIZE划分成多个page，然后再对page进行IO，简洁优美。


　　如果不作优化，Kernel Fuse对应用程序的每次page都会调用一次用户态文件系统的写操作，这样假如我们用户态的64KB的写请求，按照默认的PAGE_SIZE（4KB）可能会触发16次的用户态写，实际IO次数被放大，效率严重下降。如果采取优化，Kernel Fuse默认会每128KB才触发一次用户态文件系统写调用，当然亦可指定触发写调用的阈值。


　　优化方法：fuse挂载指定 –o big_write –o max_write=N


　　优化建议：五颗星


　　? 优化3：开启内核读缓存


　　Linux文件系统实现充分利用了内存来缓存文件数据，这样应用程序很多时候读文件其实只需从内核缓冲区拷贝数据至用户态缓冲区即可，根本不必启动磁盘IO。


　　由于Fuse的特殊性，需要严格控制数据缓存行为（看看我们前面提到的元数据缓存吧），因为可能我们实现的基于Fuse的文件系统其实是一个网络文件系统，那么如果使用内核缓存，可能就读到脏数据，因为作为用户态的你是很难控制内核的行为的。


　　不过Fuse的作者非常周到，它提供了多种挂载选项，来控制缓存行为，但友情提醒：一旦选择开启缓存，请为自己的可能读的过期数据负责。


　　优化方法：fuse挂载指定 –o kernel_cache –o auto_cache


　　顺便提一句：我们上面说的都是参数kernel_cache的行为，没有说明auto_cache的行为，留给各位读者仔细研究吧，提个醒：该选项是基于文件修改时间进行内核缓存有效性检测的优化策略。


　　优化建议：三颗星


　　? 优化4：扩大预读窗口


　　预读是在是一件有趣的事情。Linux内核通过预读改变了应用程序的原始读行为。比如应用程序发起了一个16KB的读请求，内核可能莫名其妙地读取64KB数据等。当然，它这么做肯定有其道理，简单来说：一切为了性能，为了性能的一切。另外，我会在近期推出一篇预读相关文章，详细阐述预读机制，敬请关注。


　　Fuse允许挂载用户态文件系统时指定预读窗口大小，Fuse会用该设定值作为最大的预读窗口大小，若不指定，会采用Linux默认的最大预读窗口大小128KB。但是其实如果你设置了Fuse的预读窗口超过Linux默认的128KB也是徒劳，因为VFS不允许预读窗口超过128KB限制，所以总的来说，优化的意义不大。


　　优化方法：fuse挂载指定 –o max_readahead = N


　　优化建议：一颗星


　　? 优化5：使用DirectIO取代BufferIO


　　有些时候，应用程序希望绕过OS的缓存而自己管理缓存（如数据库），这需要文件系统实现DIRECTIO方法。


　　同样，贴心的Fuse作者也为我们提供了directIO方式的读写。相比BufferIO方式，DirectIO的最大优势在于减少了数据从应用程序缓冲区拷贝至内核态的开销，对于大量顺序写的应用场景，性能可能会有一定提升。


　　当然，如果采用DirectIO，恐怕最大的问题就是read也无法使用内核缓存了，很多时候这是我们无法忍受的，常常来说，文件系统读请求会远多于写，所以，优化前望三思。


　　优化方法：fuse挂载指定 -o direct_io


　　优化建议：一颗星