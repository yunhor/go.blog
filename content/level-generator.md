这次要做的是一个关卡生成器。关卡是用 [tiled map](https://github.com/bjorn/tiled)做的。为了减轻负担，决定关卡地图自动生成。策划人员会定义一系列的规则，然后由程序员跟据相应规则生成对应的地图。

本来想直接调用tiled那个地图编辑器的库的，但稍微看了一下发现简直把人搞疯掉，UI框架的占据的东西远超过地图模型本身。代码量不少，从中找有用的相对麻烦，加上对近来对C++的鄙视，最终放弃了...

其实tiled的格式本身很简单，就是一个xml文件。反正关卡是自动生成的，我完全可以抛开地图编辑器，自已生成符合tiled格式的xml文件就行了。

用Go语言的话，数据结构到xml的序列化有包，base64编码也有包了，然后就是zlib压缩也有包的。剩下的就是搞清楚data的格式了。

具体的格式可以在 [这里](https://github.com/bjorn/tiled/wiki/TMX-Map-Format)看到，我挑其中重要的记录一下。

最高2位记录翻转。32位记录是否水平翻转，31位记录是否垂直翻转。30位记录是否对角翻转。

	const unsigned FLIPPED_HORIZONTALLY_FLAG = 0x80000000;
	const unsigned FLIPPED_VERTICALLY_FLAG   = 0x40000000;
	const unsigned FLIPPED_DIAGONALLY_FLAG   = 0x20000000;

每个格子用4字节记录的，base64解码之后应该是map_width*map_height*4大小的。数组的每个元素都是一个gid，gid是格子种类的编号。

	for (int y = 0; y < map_height; ++y) {
	  for (int x = 0; x < map_width; ++x) {
	    unsigned global_tile_id = data[tile_index] |
				      data[tile_index + 1] << 8 |
				      data[tile_index + 2] << 16 |
				      data[tile_index + 3] << 24;
	    tile_index += 4;


## 地图校验

生成tmx格式的地图并不难。难点是验证生成的地图是有解的。这个东西比较麻烦，开始想过一种思路是生成地图的时候直接按某种规则去生成有解的地图。但是由于卷屏，障碍物，特殊道具等，使得这个实现起来不太可行。

然后就只好用最笨的方式了，随机生成地图，事后验证地图有解。模拟点击事件进行校验，暴力搜索所有的状态。按深度优先的方式去搜索，只要找到一个解就退出。内存开销方面，只有在状态被遍历到的时候才会生成新的状态，内存使用量也就是搜索深度乘以一层的分支数。内存应该问题不大。

出现的一个问题是时间复杂度。假设只是7*9的格式，则一层的分支数是63，地图无解的情况下需要遍历所有可能的状态，于是会出现63^n条路径需要遍历。而且每一次状态进行验证时，也涉及到了一个mark-sweep，是一个深度或广度优先的算法。随便就到了阶乘的复杂度，时间复杂度太高了，几乎不可用。3*7的无解地图，足足用了10多秒才能确定它是无解的。5*7的无解的地图跑了几分钟都没出结果。

时间问题确实是目前的一个大难题，想到的一些解决思路。一方面是做限时。这个问题很明显，有解的情况只要找到一个解就可以退出，因此速度很快。而无解情况要遍历所有的情况才能确定，因此会很慢。限定一个时间，如果没有找到一个解，就退出，抛弃这张地图，假定它是无解的。反正我的需求是找有解的地图。

另一方面就是做一些启发式的搜索了，优先挑更可能的路径进行搜索，比如贪心算法，每次找最大的一块进行点击。这个更接近用户行为，这样找到的解也更好理解一些。否则出现某张地图有解，但用户试了很多遍都无法通关的情况，他们要么买道具，更可能就是删游戏了。

接下来还能做的一些优化就是并行方面考虑。