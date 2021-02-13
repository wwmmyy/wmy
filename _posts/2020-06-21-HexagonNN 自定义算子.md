---
layout:     post
title:      HexagonNN 自定义算子
subtitle:   
date:       2020-06-21
author:     WMY
header-img: img/15.jpg
catalog: true
tags:
    - 工作
---

# 背景:

有时候深度学习 开发者可能需要根据自己的需求增加新的算子， 本篇将会介绍如何增加一个运行在DSP上的新的算子，将该算子编译到libhexagon_nn_skel.so中，并通过一个android环境下如何调用该算子。

# Overview:

源码下载：下载最新的高通hexagon nnlib 源码： git clone https://source.codeaurora.org/quic/hexagon_nn/nnlib/  或者下载完整的Qualcomm Hexagon_SDK 下载完成后nnlib源码在：Hexagon_SDK\xxx（sdk版本号）\libs\hexagon_nn\ 路径下;

* 算子简称为Ops 存放在 `$HEXAGON_NN/hexagon/ops/src/` 路径下，每个算子必须遵循如下规范:
    * A struct of type `nn_node_ops` which provides the following functions:
        1) .execute called by `hexagon_nn_execute()`, operates on tensors
        2) .check called during setup, validates number of in/out tensors
        3) .ctor constructor, typically just 'node_alloc_common'
        4) .dtor destructor, typically just 'node_free_common'
    * Implementations of the `.execute` and `.check` functions
    * The name of the `nn_node_ops` struct should be of the form: `struct nn_node_ops nn_ops_for_<OP_NAME>`

* Files added in `$HEXAGON_NN/hexagon/ops/src/` should be referenced in `$HEXAGON_NN/hexagon/files.mak`

* Ops defined as 'struct nn_node_ops nn_ops_for_<OP_NAME>' should be declared as `DEF_OP(<OP_NAME>)` in `$HEXAGON_NN/interface/ops.def`

* Once declared and defined, you can instantiate your new ops as `OP_<OP_NAME>`. For example, like the following:
```
hexagon_nn_append_node(gid,nid,OP_AddConst_8,NN_PAD_NA,NULL,0,NULL,0);
```

# 详细案例：

1)按照示例，创建一个`op_addconst.c` 的文件.该文件实现了一维数组跟一个常量相加，按照以上描述，该文件里面必须创一个`nn_node_ops` 结构体, 并实现该结构体中的 `.execute` 和`.check`方法：
部分代码摘录如下：

	static int addconst_execute(struct nn_node *self, struct nn_graph *nn)
	{
		...
		// Read the constant from our scalar input
		unsigned char constant = ((char*)b_tensor->data)[0];
	
		// Add constant to tensor
		int i;
		char *in = (char *) a_tensor->data;
		char *out = (char *) out_tensor->data;
		for (i=0; i<out_tensor->data_size; i++) {
			*(out++) = *(in++) + constant;
		}
	
		// Return 0 for success, 1 for failure of node.
		return 0;
	}
	
	// .check functions validate the number of inputs and outputs for the op-type.
	// Return 0=success, 1=failure
	static int addconst_check(struct nn_node *self, struct nn_graph *nn)
	{
		if (self->n_inputs != 2) return errlog(nn,"Wrong # inputs");
		if (self->n_outputs != 1) return errlog(nn,"Wrong # outputs");
		return 0;
	}
	
	// This struct says how the "OP_AddConst_8" will work.
	struct nn_node_ops nn_ops_for_AddConst_8 = {
		.execute = addconst_execute,
		.check = addconst_check,
		.ctor = node_alloc_common,
		.dtor = node_free_common,
	};


该新增的算子文件需要存放在 $HEXAGON_NN/hexagon/ops/src/ 路径下；

2) 添加在新增文件`op_addconst.c`到 `$HEXAGON_NN/hexagon/hexagon_nn_c_srcs.txt` 编译文件中，这样在编译时就会将其编译进去

3) 将新增的算子名称添加到 $HEXAGON_NN/interface/ops.def 声明文件中，这样在实际调用中才能被调用到。

在ops.def 的尾部添加  `DEF_OP_WREF(AddConst_8)` 即可；

*Note: `DEF_OP_WREF`是一个宏定义在ops.def 文件中：

		#ifndef DEF_OP_WREF
		#define DEF_OP_WREF(NAME) DEF_OP(NAME) DEF_OP(NAME##_ref)
		#define __SELF_DEF_OP_WREF
		#endif
	
	`DEF_OP` 也是一个宏，在$HEXAGON_NN/interface/hexagon_nn_ops.h 中的定义为：
	
		#define DEF_OP(NAME,...) OP_##NAME,
		typedef enum op_type_enum {
		#include "ops.def"
			NN_OPS_MAX
		} op_type;
		#undef DEF_OP

可以看出声明在ops.def中的算子，实际上会被展开为'OP_##NAME',同时保存在 op_type的枚举变量中。

此外 `DEF_OP`  在 $HEXAGON_NN\ops\src\optab.c 中的宏定义为：

	#define DEF_OP(NAME,...) extern struct nn_node_ops nn_ops_for_##NAME;
	#include "../../interface/ops.def"
	#undef DEF_OP
	
	#define DEF_OP(NAME,...) [OP_##NAME] = &nn_ops_for_##NAME,
	struct nn_node_ops *optab[NN_OPS_MAX] = {
	#include "../../interface/ops.def"
	};

因此声明在ops.def中的算子同时也会被展开为 [OP_##NAME] = &nn_ops_for_##NAME， 类似hashMap的方式保存在optab数组中。 对于上述自定义算子而言 &nn_ops_for_##NAME对应了上述`op_addconst.c`中的 `nn_ops_for_AddConst_8`方法，通过 optab[OP_AddConst_8]即可获取该方法名称 `nn_ops_for_AddConst_8`.



此外，了便于阅读，可以将自定义的算子假如声明文档中 `$HEXAGON_NN/docs/ops.txt`, 添加新增算子说明如下:

    AddConst_8:
    	input 0: tensor (uint8)
	input 1: scalar (uint8)
    	output 0: tensor (uint8)
    	Adds scalar value to input tensor, producing output tensor

4) 现在开始再Hexagon_SDK环境下编译该工程，会生成包含自定义算子的库文件'libhexagon_nn_skel.so'
    ```
    $ cd $HEXAGON_NN && make VERBOSE=1 V=hexagon_ReleaseG_dynamic_toolv83_v66 tree
    ```
5) 创建测试Case执行新增自定义算子

	创建测试文件“005-addconst.c” 对应的网络结构如下：

	// The structure of our ADD network looks like this, and has
	//   three layers and four nodes:
	//
	//          [ 42 ] <------Constant
	//          "layer1a"
	//           0x1001a
	//                \.
	// INPUT (2x2) --> AddConst --> OUTPUT (2x2)
	//   "layer0"    "layer1"       "layer2"
	//    0x1000      0x1001         0x1002

在如上文件中 调用自定义算子所对应的tensor节点声明为：

    err |= hexagon_nn_append_node(
            graph_id,
            0x1001,
            OP_AddConst_8, //即为自定义算子方法名
            NN_PAD_NA,
            layer1_input_list,
            2,
            layer1_output_list,
            1
            );

     if ((err = hexagon_nn_prepare(graph_id)) != 0) {
                printf("Whoops... Cannot prepare: %d\n", err);
                goto TEARDOWN;
        }

        uint8_t *output;
        if ((output = rpcmem_alloc(
                     ION_HEAP_ID_SYSTEM,
                     RPCMEM_DEFAULT_FLAGS,
                     OUT_SIZE)
                    ) == NULL) {
                printf("Whoops... Cannot malloc for outputs\n");
                err=1;
                goto TEARDOWN;
        }

        uint8_t *input;
        if ((input = rpcmem_alloc(
                     ION_HEAP_ID_SYSTEM,
                     RPCMEM_DEFAULT_FLAGS,
                     IN_SIZE)
                    ) == NULL) {
                printf("Whoops... Cannot malloc for inputs\n");
                err=1;
                goto TEARDOWN;
        }

        if ((err = hexagon_nn_execute(
                     graph_id,
                     IN_BATCH,
                     IN_HEIGHT,
                     IN_WIDTH,
                     IN_DEPTH,
                     (const uint8_t *)input, // Pointer to input data
                     IN_SIZE,                // How many total bytes of input?
                     &out_batches,
                     &out_height,
                     &out_width,
                     &out_depth,
                     (uint8_t *)output,      // Pointer to output buffer
                     OUT_SIZE,               // Max size of output buffer
                     &out_data_size)         // Actual size used for output
                    ) != 0) {

                printf("Whoops... run failed: %d\n",err);
                goto TEARDOWN;
        }
    
 
将 测试文件在android.min中进行声明，并执行编译命令，生成对应的测试文件：

    $ make V=android_ReleaseG tree VERBOSE=1 CDSP_FLAG=1


6) 将生成的文件执行以下命令运行:

    $ adb shell mkdir -p /vendor/bin/nn_tutorials
    $ adb push android_ReleaseG\ship\005-addconst /vendor/bin/nn_tutorials
    $ adb push $HEXAGON_NN/hexagon_ReleaseG_dynamic_toolv83_v66/ship/libhexagon_nn_skel.so /vendor/lib/rfsa/adsp
    $ adb shell chmod 755 /vendor/bin/nn_tutorials/005-addconst
    $ adb shell /vendor/bin/nn_tutorials/005-addconst
 
该测试Case接受一个int型的 2x2 tensor输入，例如输入以下参数会有对应返回值:

$ 005-addconst 0 1 2 3
    [ 42 43 ]
    [ 44 45 ]

如上的testcase是如何关联调用到自定义算子的呢，接着我们看下每个node节点定义：

	int hexagon_nn_append_node(
		nn_id id,
		uint32_t node_id,
		uint32_t operation,
		padding_type padding,
		const hexagon_nn_input *inputs,
		uint32_t n_inputs,
		const hexagon_nn_output *outputs,
		uint32_t n_outputs);
	
	int hexagon_nn_append_node(...)
	{
		...
		return do_append_node(
			graph,
			node_id,
			operation,
			padding,
			num_inputs,
			num_outputs,
			inputs,
			outputs);
	}
	
	int do_append_node(...)
	{
		...
		struct nn_node *node;
		if ((node = optab[operation]->ctor( //注意下面会对此函数重点追踪分析
			     nn,
			     node_id,
			     operation,
			     padding,
			     num_inputs,
			     num_outputs,
			     inputs,
			     outputs)) == NULL) {
			return errlog(nn,"node id=0x%x ctor fail",node_id);
		}
		...
		node_append(nn,node);
		..
	}


根据上文可知 `DEF_OP`  在 $HEXAGON_NN\ops\src\optab.c 中的宏定义为：

	#define DEF_OP(NAME,...) [OP_##NAME] = &nn_ops_for_##NAME,
	struct nn_node_ops *optab[NN_OPS_MAX] = {
	#include "../../interface/ops.def"
	};

ops.def 中是各种算子的声明信息：
	
	...
	DEF_OP(QuantizedPadForConv_8_d32)
	DEF_OP(QuantizedPadForConv_u16_d32)
	DEF_OP(LSHProjection)
	DEF_OP(Requantize_8to8)
	DEF_OP(InputSupernode_16x16p16to16_outd32)
	DEF_OP(InputSupernode_16x16p32to16_outd32)
	DEF_OP(Requantize_8to8_d32)
	
	DEF_OP_WREF(QuantizedCorrelation1d_8x8to8)
	DEF_OP_WREF(AddConst_8)
	...


因此通过以上宏展开后，保存在optab数组中的信息有： ```[OP_AddConst_8] = &nn_ops_for_AddConst_8```
而 nn_ops_for_AddConst_8的定义为：
	
	// This struct says how the "OP_AddConst_8" will work.
	struct nn_node_ops nn_ops_for_AddConst_8 = {
		.execute = addconst_execute,
		.check = addconst_check,
		.ctor = node_alloc_common,
		.dtor = node_free_common,
	};

因此如上代码中```optab[operation]->ctor(...)``` 对应的执行函数为: 'node_alloc_common`
	
	struct nn_node *node_alloc_common(
	        struct nn_graph *nn,
	        uint32_t node_id,
	        op_type operation,
	        padding_type padding,
	        uint32_t num_inputs,
	        uint32_t num_outputs,
	        const struct input *inputs,
	        const struct output *outputs)
	{
	        struct nn_node *newnode;
	        ...
	        if ((newnode = alloc_node(node_id,operation,padding)) == NULL) {
	                errlog(nn,"common alloc id %x malloc fail",node_id);
	                return NULL;
	        }
	        ...
	        return newnode;
	}
	
	struct nn_node *alloc_node(uint32_t node_id, 
		op_type operation, padding_type padding)
	{
		struct nn_node *newnode;
		if ((newnode = nn_malloc(sizeof(*newnode))) == NULL) {
			return newnode;
		}
		newnode->node_id = node_id;
		newnode->ops = optab[operation];//每个node对应的 nn_node_ops 也会记录在对应的 node中
		....

从可看出 `newnode->ops = optab[operation]` 针对自定义的算子它保存了 `optab[OP_AddConst_8]` 对应 `&nn_ops_for_AddConst_8`即：

	struct nn_node_ops nn_ops_for_AddConst_8 = {
		.execute = addconst_execute,//该方法对应具体的算法实现
		.check = addconst_check,
		.ctor = node_alloc_common,
		.dtor = node_free_common,
	};

其中新增节点 `nn_node` 定义为：

	struct nn_node {
		struct nn_node_ops *ops;	// Operations for this NN //注意以上测试用例中`OP_AddConst_8`会被保存在这里，接下来执行阶段会遍历每个node调用该结构体中的算子方法执行
		const struct tensor **inputs;	// Inputs
		struct tensor **outputs;	// Outputs
		struct input *input_refs;	// References to node outputs
		struct output *output_defs;	// Output definitions
		uint32_t n_inputs;		// Number of inputs
		uint32_t n_outputs;		// Number of outputs
		noderefhash_set_t noderefhash;     // 'or' of noderefhash_mask( input_refs[i].src_id ) for all inputs
		uint32_t node_type;		// node type
		uint32_t node_id;		// node ID
		padding_type padding;		// kind of padding
		struct nn_node *next;		// ptr to next node
		void *opaque;			// whatever the node wants to put here
		uint32_t flags;
		uint32_t executions;		// how many times the node executes
		uint32_t refs;			// time op was referenced by any output
		uint64_t perfcounter;		// performance counter
		uint64_t iter_cycles;		// cycles consumed in last execution
	        struct udo_node_info udo_info;  // udo function and data pointers
	};

	struct nn_node_ops {
		int (*execute)(struct nn_node *self, struct nn_graph *nn); //每个node对应算子的执行算法
		int (*check)(struct nn_node *self, struct nn_graph *nn);
		struct nn_node *(*ctor)(
			struct nn_graph *nn,
			....

如上每个新增node及其对应的 operation 都会被保存在网络中。

接下来看在执行阶段在执行阶段如何调用到这些node节点及其对应的算子算法：

	
	int hexagon_nn_execute(...)
	{
		....
		ret = hexagon_nn_execute_new(...);
		...
	}
	
	int hexagon_nn_execute_new(
		nn_id_t id,
		const hexagon_nn_tensordef *inputs,
		uint32_t n_inputs,
		hexagon_nn_tensordef *outputs,
		uint32_t n_outputs)
	{
	        execute_basic_info exe_info;
	        return execute_inner(id, inputs, n_inputs, outputs, n_outputs, &exe_info);
	}
	
	static int execute_inner(...)
	{
	        ....
	        ret = do_execute(graph, exe_info);
	        ....
	}

这里会遍历所有的tensor 节点并执行:
	
	int do_execute(struct nn_graph *nn, execute_basic_info* exe_info)
	{
		..... 
		struct nn_node *node;
		do{
		for (node = start_node; node != NULL; node = next_node) {
			logmsg(nn,4,"do_execute(): node=%p id=%x, next at %p",node,node->node_id, node->next);
			//execute_check_src_canaries(nn,node);
			//execute_set_canaries(nn,node);
			perf_start = nn_os_get_perfcount(nn);
			pcycle_node = nn_os_get_cycles(nn);
			nn_scratch_reset(nn);
			/* for (int j = 0; j < node->n_inputs; j++) {
				print_tensor(node->inputs[j],"in");
			}*/
			if ((err = node->ops->execute(node,nn)) != 0) { //这里会遍历每个node及其对应的nn_node_ops，执行对应的算子算法
	                        exe_info->exe_failure_node_id = node->node_id;
	                        exe_info->exe_failure_node_op_type = node->node_type;
	                        if (node->node_type==NN_OPS_MAX) {
	                                exe_info->result = NN_EXECUTE_UDO_ERROR;
	                        } else if (err==NN_EXECUTE_BUFFER_SIZE_ERROR) { 
	                                exe_info->result = err;
	                        } else {
	                                exe_info->result = NN_EXECUTE_ERROR;
	                        }
				errlog(nn,"execute() failed on node id=%x err=%d",node->node_id,err);
				goto quit;
			}
			.....
		} // for node list
		}while( nn_batchseqstate_loop_update( &nn->batchseq )); // batch seq loop
		} // for ITERS
	       ....
	}

因此根据以上分析遍历执行 `node->ops->execute` 会调用到 `nn_ops_for_AddConst_8` 算子及具体实现算法 `addconst_execute`；
