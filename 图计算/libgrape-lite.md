#### libgrape-lite

##### 大致工作流程

> 参考：run_app.h/.cpp，sssp.h等

- 初始化

  ```cpp
  InitMPIComm();
  // CommSpec records the mappings of fragments, workers, and the threads(tasks) in each worker.
  grape::CommSpec comm_spec;
  comm_spec.Init(MPI_COMM_WORLD);
  ```

- 运行

  *例如运行SSSP*

  ```cpp
  // 平行引擎参数
  grape::ParallelEngineSpec spec = MultiProcessSpec(comm_spec, __AFFINITY__);
  // 设置线程数
  spec.cpu_list.resize(spec.thread_num);
  // 所有fragment数量，可以说是节点数量
  int fnum = comm_spec.fnum();
  
  // 传入SSSP作为APP_T
  CreateAndQuery<OID_T, VID_T, VDATA_T, double, LoadStrategy::kOnlyOut, SSSP, OID_T>(comm_spec, out_prefix, fnum, spec, FLAGS_sssp_source);
  
  // 加载图的参数设置
  grape::LoadGraphSpec graph_spec = DefaultLoadGraphSpec();
  graph_spec.set_directed(FLAGS_directed);
  
  // 加载图
  std::shared_ptr<FRAG_T> fragment = grape::LoadGraph<FRAG_T>(FLAGS_efile, FLAGS_vfile, comm_spec, graph_spec);
  auto app = std::make_shared<AppType>();
  
  // 生成worker，工作
  auto worker = APP_T::CreateWorker(app, fragment);
  worker->Init(comm_spec, spec);
  worker->Query(std::forward<Args>(args)...);
  
  std::ofstream ostream;
  std::string output_path = grape::GetResultFilename(out_prefix, fragment->fid());
  ostream.open(output_path);
  worker->Output(ostream);
  ostream.close();
  
  worker->Finalize();
  ```

  worker的Query方法

  ```cpp
  // 等待所有的工作节点到达这个地方，再继续
  MPI_Barrier(comm_spec_.comm());
  // 初始化，比如存储sssp中的source_id号（在args里）
  context_->Init(messages_, std::forward<Args>(args)...);
  // 待述
  processMutation();
  int round = 0;
  
  messages_.Start();
  
  messages_.StartARound();
  // 开始PEval
  runPEval();
  processMutation();
  messages_.FinishARound();
  
  int step = 1;
  // 循环处理IncEval
  while (!messages_.ToTerminate()) {
    round++;
    messages_.StartARound();
    runIncEval();
    processMutation();
    messages_.FinishARound();
  
    ++step;
  }
  
  // 等待所有的节点过来，一起结束
  MPI_Barrier(comm_spec_.comm());
  messages_.Finalize();
  ```

  sssp的PEval和IncVal方法

  ```cpp
  /*
  * curr_modified：当前更改的点，比如接收到数据，更改了value的点
  * next_modified：下个更改的点，受当前影响到的点，比如更改了outer的点值
  */
  void PEval(const fragment_t& frag, context_t& ctx, message_manager_t& messages) {
      // 一个进程管理一个fragment，可以再分为多个线程共同处理这一个fragment
      messages.InitChannels(thread_num());
  
      // sssp中的源点
      vertex_t source;
      bool native_source = frag.GetInnerVertex(ctx.source_id, source); 
      // 清空
      ctx.next_modified.ParallelClear(GetThreadPool());
  
      // Get the channel. Messages assigned to this channel will be sent by the
      // message manager in parallel with the evaluation process.
      auto& channel_0 = messages.Channels()[0];
  
      if (native_source) {
        ctx.partial_result[source] = 0;
        auto es = frag.GetOutgoingAdjList(source);
        for (auto& e : es) {
          vertex_t v = e.get_neighbor();
          ctx.partial_result[v] =
              std::min(ctx.partial_result[v], static_cast<double>(e.get_data()));
          if (frag.IsOuterVertex(v)) {
            // put the message to the channel.
            channel_0.SyncStateOnOuterVertex<fragment_t, double>(
                frag, v, ctx.partial_result[v]);
          } else {
            ctx.next_modified.Insert(v);
          }
        }
      }
  
      // 本轮结束，推进
      messages.ForceContinue();
      // 让curr_modified存放本轮修改过的顶点
      ctx.next_modified.Swap(ctx.curr_modified);
  }
  
  void IncEval(const fragment_t& frag, context_t& ctx,
                 message_manager_t& messages) {
      auto inner_vertices = frag.InnerVertices();
      auto& channels = messages.Channels();
      ctx.next_modified.ParallelClear(GetThreadPool());
  
      // parallel process and reduce the received messages
      // u是接收点，是本frag的inner点，这里没看到发送点是哪个的
      messages.ParallelProcess<fragment_t, double>(
          thread_num(), frag, [&ctx](int tid, vertex_t u, double msg) {
            if (ctx.partial_result[u] > msg) {
              atomic_min(ctx.partial_result[u], msg);
              ctx.curr_modified.Insert(u);
            }
          });
  
      // incremental evaluation.
      // 遍历所有内部节点，更新其邻接点。邻接点可能在inner里，也可能在outer里
      // 更改的邻接点记录在next_modified内
      ForEach(ctx.curr_modified, inner_vertices,
              [&frag, &ctx](int tid, vertex_t v) {
                double distv = ctx.partial_result[v];
                auto es = frag.GetOutgoingAdjList(v);
                for (auto& e : es) {
                  vertex_t u = e.get_neighbor();
                  double ndistu = distv + e.get_data();
                  if (ndistu < ctx.partial_result[u]) {
                    atomic_min(ctx.partial_result[u], ndistu);
                    ctx.next_modified.Insert(u);
                  }
                }
              });
  
      // put messages into channels corresponding to the destination fragments.
      // 在next_modified中挑选outer点，将更改的数据同步给对应的frag
       // 即同步mirror点状态给outer点
      auto outer_vertices = frag.OuterVertices();
      ForEach(ctx.next_modified, outer_vertices,
              [&channels, &frag, &ctx](int tid, vertex_t v) {
                channels[tid].SyncStateOnOuterVertex<fragment_t, double>(
                    frag, v, ctx.partial_result[v]);
              });
  
      if (!ctx.next_modified.PartialEmpty(
            frag.Vertices().begin_value(),
            frag.Vertices().begin_value() + frag.GetInnerVerticesNum())) {
        messages.ForceContinue();
      }
  
      ctx.next_modified.Swap(ctx.curr_modified);
  }
  ```

- 结束

  ```cpp
  FinalizeMPIComm();
  ```

##### 图加载和构建

##### 从文件加载

```cpp
// loader.h
std::shared_ptr<FRAG_T> LoadGraph(efile, vfile, comm_spec, spec) {
  std::unique_ptr<EVFragmentLoader<FRAG_T, IOADAPTOR_T, LINE_PARSER_T>> loader(
      new EVFragmentLoader<FRAG_T, IOADAPTOR_T, LINE_PARSER_T>(comm_spec));
  return loader->LoadFragment(efile, vfile, spec);
}

// ev_fragment_loader.h
std::shared_ptr<fragment_t> LoadFragment(const std::string& efile,
                                           const std::string& vfile,
                                           const LoadGraphSpec& spec) {
    std::shared_ptr<fragment_t> fragment(nullptr);
    // 加载节点
    auto io_adaptor = std::unique_ptr<IOADAPTOR_T>(new IOADAPTOR_T(vfile));
    while (io_adaptor->ReadLine(line)) {
        ++line_no;
        // 解析出一行，这里只解析了vid和一个data值
        line_parser_.LineParserForVFile(line, vertex_id, v_data);	// 1
        // id_list: vfile中的vid; vdata_list: vfile中的v的data数据
        id_list.push_back(vertex_id);
        vdata_list.push_back(v_data);
    }
    io_adaptor->Close();
    // partitioner_t: fragment_t::vertex_map_t::partitioner_t
    partitioner_t partitioner(comm_spec_.fnum(), id_list);		// 1.1
    // basic_fragment_loader_: BasicFragmentLoader<fragment_t, io_adaptor_t>
    // 设置分配器
    basic_fragment_loader_.SetPartitioner(std::move(partitioner));
    // 设置平衡参数
    basic_fragment_loader_.SetRebalance(spec.rebalance,
                                        spec.rebalance_vertex_factor);
    basic_fragment_loader_.Start();			// 1.2
    // 添加点到basic_fragment_loader_中
    size_t vnum = id_list.size();
    for (size_t i = 0; i < vnum; ++i) {
    	basic_fragment_loader_.AddVertex(id_list[i], vdata_list[i]);	// 2
    }
    
    // 加载边
    auto io_adaptor = std::unique_ptr<IOADAPTOR_T>(new IOADAPTOR_T(std::string(efile)));
    // 只是设置了可以partial_read
    io_adaptor->SetPartialRead(comm_spec_.worker_id(), comm_spec_.worker_num());
    io_adaptor->Open();
    // parse出3个字段
    line_parser_.LineParserForEFile(line, src, dst, e_data);
    basic_fragment_loader_.AddEdge(src, dst, e_data);
    io_adaptor->Close();
    // 构建frag
    basic_fragment_loader_.ConstructFragment(fragment, spec.directed);	// 1.3
    return fragment;
}

// tsv_line_parser.h
void LineParserForVFile(const std::string& line, OID_T& u, VDATA_T& u_data) {	// [1]
	this->LineParserForEverything(line, u, u_data);	// F3
}
// F1
const char* LineParserForEverything(const std::string& line,
                                             Ts&... vals) {
    return this->LineParserForEverything(
        line.c_str(),
        std::forward<typename std::add_lvalue_reference<Ts>::type>(vals)...);
}
// F2, 解析head字符串，返回下一个位置
const char* LineParserForEverything(const char* head, T& val) {
    return internal::match(		// 2
        head, std::forward<typename std::add_lvalue_reference<T>::type>(val));
}
// F3
const char* LineParserForEverything(const char* head, T& val, Ts&... vals) {
    // 解析出head第一个字段（点id），向后移动一个
    const char* next_head = internal::match(
        head, std::forward<typename std::add_lvalue_reference<T>::type>(val));
    // 执行F2
    return this->LineParserForEverything(
        next_head,
        std::forward<typename std::add_lvalue_reference<Ts>::type>(vals)...);
}

// line_parser_base.h
// 提取出str最前面一个元素存入r，并移动str到下一个位置
/* @T: 支持多种类型:
   整数，浮点
   std::string
   grape::EmptyType
*/
const char* match(char const* str, T& r, char const* end = nullptr);
// r为string的解析方法。分隔符：' ', '\t'
const char* match<std::string>(char const* str, std::string& r, char const* end) {
    int nlen1 = 0, nlen2 = 0;
	// t去头部杂乱符号，相当于trim
    while (str + nlen1 != end && str[nlen1] &&
         (str[nlen1] == '\n' || str[nlen1] == ' ' || str[nlen1] == '\t')) {
    	nlen1 += 1;
    }
    nlen2 = nlen1;
    while (str + nlen2 != end && str[nlen2] &&
         (str[nlen2] != '\n' && str[nlen2] != ' ' && str[nlen2] != '\t')) {
    	nlen2 += 1;
    }
    // nlen1: 头元素的起始位置；nlen2：头元素的结束位置 + 1
    r = std::string(str + nlen1, nlen2 - nlen1);
    return str + nlen2;
}

// partitioner.h
void SegmentedPartitioner(size_t frag_num, std::vector<OID_T>& oid_list) {	// [1.1]
    fnum_ = frag_num;
    size_t vnum = oid_list.size();
    // 平均分，至少分一个
    size_t frag_vnum = (vnum + fnum_ - 1) / fnum_;
    o2f_.reserve(vnum);
    for (size_t i = 0; i < vnum; ++i) {
        fid_t fid = static_cast<fid_t>(i / frag_vnum);
        o2f_.emplace(oid_list[i], fid);
    }
}

// basic_fragment_loader.h
void BasicFragmentLoader::AddVertex(const oid_t& id, const vdata_t& data) {	// [2]
    // internal_oid_t: InternalOID<oid_t>::type
    internal_oid_t internal_id(id);		// 3
    auto& partitioner = vm_ptr_->GetPartitioner();
    fid_t fid = partitioner.GetPartitionId(internal_id);
    // vertices_to_frag_: std::vector<ShuffleOut<internal_oid_t, vdata_t>>
    // fid: [ShuffleOut]; 将内部格式的{vid, data}发给对应的worker
    vertices_to_frag_[fid].Emplace(internal_id, data);		// 2.1
}
// 开启点和边的接收线程
void Start() {			// [1.2]
    vertex_recv_thread_ = std::thread(&BasicFragmentLoader::vertexRecvRoutine, this);
    edge_recv_thread_ = std::thread(&BasicFragmentLoader::edgeRecvRoutine, this);
    recv_thread_running_ = true;
}
// 接收点信息，点和边信息是多个节点一起解析的，互相同步
void vertexRecvRoutine() {
    ShuffleIn<internal_oid_t, vdata_t> data_in;
    data_in.Init(comm_spec_.fnum(), comm_spec_.comm(), vertex_tag);
    fid_t dst_fid;
    int src_worker_id;
    while (!data_in.Finished()) {
        src_worker_id = data_in.Recv(dst_fid);
        if (src_worker_id == -1) {
        	break;
        }
        got_vertices_.emplace_back(std::move(data_in.buffers()));
        data_in.Clear();
    }
}
void edgeRecvRoutine() {
    ShuffleIn<internal_oid_t, internal_oid_t, edata_t> data_in;
    data_in.Init(comm_spec_.fnum(), comm_spec_.comm(), edge_tag);
    fid_t dst_fid;
    int src_worker_id;
    while (!data_in.Finished()) {
        src_worker_id = data_in.Recv(dst_fid);
        if (src_worker_id == -1) {
        	break;
        }
        CHECK_EQ(dst_fid, comm_spec_.fid());
        got_edges_.emplace_back(std::move(data_in.buffers()));
        data_in.Clear();
    }
}

// shuffle.h
// 将解析出的点和frag的对应关系同步出去
void Emplace(const TYPES&... rest) {		// [2.1]
    buffers_.Emplace(rest...);
    ++current_size_;
    if (comm_disabled_) {
		return;
    }
    if (current_size_ >= chunk_size_) {
        issue();
        Clear();
    }
}
void issue() {
    frag_shuffle_header header(current_size_, dst_frag_id_);
    sync_comm::Send<frag_shuffle_header>(header, dst_worker_id_, tag_, comm_);

    if (current_size_) {
		buffers_.SendTo(dst_worker_id_, tag_, comm_);
    }
}

// types.h
struct InternalOID {		// [3]
    using type = T;
    static type ToInternal(const T& val) { return val; }
    static T FromInternal(const type& val) { return val; }
};
// 看起来只是对string类型进行了转换
struct InternalOID<std::string> {
    using type = nonstd::string_view;
    static type ToInternal(const std::string& val) {
    	return nonstd::string_view(val.data(), val.size());
    }
	static std::string FromInternal(const type& val) { return std::string(val); }
};

// basic_fragment_loader.h
void ConstructFragment(std::shared_ptr<fragment_t>& fragment, bool directed) {	// [1.3]
	// 将待发送的点信息发出去
    for (auto& va : vertices_to_frag_) {
		va.Flush();
    }
    for (auto& ea : edges_to_frag_) {
		ea.Flush();
    }
    // 等接收线程结束
    vertex_recv_thread_.join();
    edge_recv_thread_.join();
    recv_thread_running_ = false;

    MPI_Barrier(comm_spec_.comm());
    // got_vertices_存放的是接收到的点信息；这里再将本frag里的点信息放进去
    got_vertices_.emplace_back(
        std::move(vertices_to_frag_[comm_spec_.fid()].buffers()));
    vertices_to_frag_[comm_spec_.fid()].Clear();
    got_edges_.emplace_back(
        std::move(edges_to_frag_[comm_spec_.fid()].buffers()));
    
    vm_ptr_->Init();
    // builder: GlobalVertexMapBuilder
    auto builder = vm_ptr_->GetLocalBuilder();
    // 将本frag的点加进来，用IdIndexer存
    for (auto& buffers : got_vertices_) {
        foreach_helper(
            buffers,
            [&builder](const internal_oid_t& id) { builder.add_vertex(id); },
            make_index_sequence<1>{});
    }
    // 边的
    for (auto& buffers : got_edges_) {
        foreach_helper(
            buffers,
            [&builder](const internal_oid_t& src, const internal_oid_t& dst) {
                builder.add_vertex(src);
                builder.add_vertex(dst);
            },
            make_index_sequence<2>{});
    }
    // 同步点边信息
    builder.finish(*vm_ptr_);
    
    // 存点的vdata进processed_vertices_
    // processed_vertices_: std::vector<internal::Vertex<vid_t, vdata_t>>
    if (!std::is_same<vdata_t, EmptyType>::value) {
        for (auto& buffers : got_vertices_) {
            foreach_rval(buffers, [this](internal_oid_t&& id, vdata_t&& data) {
                vid_t gid;
                CHECK(vm_ptr_->_GetGid(id, gid));
                processed_vertices_.emplace_back(gid, std::move(data));
            });
        }
    }
    // 边的
    for (auto& buffers : got_edges_) {
        foreach_rval(buffers, [this](internal_oid_t&& src, internal_oid_t&& dst,
                                   edata_t&& data) {
            vid_t src_gid, dst_gid;
            CHECK(vm_ptr_->_GetGid(src, src_gid));
            CHECK(vm_ptr_->_GetGid(dst, dst_gid));
            processed_edges_.emplace_back(src_gid, dst_gid, std::move(data));
        });
    }
    // 初始化本frag
    fragment = std::shared_ptr<fragment_t>(new fragment_t(vm_ptr_));
    fragment->Init(comm_spec_.fid(), 
                   directed, 
                   processed_vertices_, 
                   processed_edges_);
    // 初始化outer的点的data
    if (!std::is_same<vdata_t, EmptyType>::value) {
		initOuterVertexData(fragment);
    }
}


```

- `std::forward`：通常是用于完美转发的，它会将输入的参数原封不动地传递到下一个函数中，这个“原封不动”指的是，如果输入的参数是左值，那么传递给下一个函数的参数的也是左值；如果输入的参数是右值，那么传递给下一个函数的参数的也是右值。

##### 零碎

- `Vertex<VID_T>`: 其value成员存放的是id，是local id

- `frag.GetOuterVertexGid(v) / GetInnerVertexGid(v)`: 获取外部/内部节点的全局id

- local id和gid都不是oid（文件里的）

  - vid = gid = `[fid][local id]`；位数等于`sizeof(VID_T) * 8`

- ```cpp
  fnum_ = worker_num_;
  fid_ = worker_id_;
  ```
