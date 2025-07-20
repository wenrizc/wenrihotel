-- 1. 系统用户表 (sys_user)
CREATE TABLE sys_user (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    username VARCHAR(255), -- 用户名
    password VARCHAR(255), -- 密码
    status INT -- 状态：1-正常，0-离线
);

COMMENT ON TABLE sys_user IS '系统用户表';
COMMENT ON COLUMN sys_user.id IS '主键ID';
COMMENT ON COLUMN sys_user.created_at IS '创建时间';
COMMENT ON COLUMN sys_user.updated_at IS '更新时间';
COMMENT ON COLUMN sys_user.username IS '用户名';
COMMENT ON COLUMN sys_user.password IS '密码';
COMMENT ON COLUMN sys_user.status IS '状态：1-正常，0-离线';


-- 2. 文档元数据表 (documents)
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    source_uri TEXT, -- 原始文件的来源路径，可以是本地文件路径或网络URL
    file_name VARCHAR(255), -- 原始文件名
    file_type VARCHAR(50), -- 由系统智能识别的文件类型，如 'PDF', 'EPUB', 'TXT'
    file_size BIGINT, -- 文件大小，单位为字节
    status VARCHAR(50), -- 当前文档的处理状态，如 'PENDING', 'PARSED', 'CHUNKED', 'INDEXED', 'ERROR'
    metadata_json TEXT -- 存储其他从文档中提取的、非结构化的元数据，如发布日期、出版社等
);

COMMENT ON TABLE documents IS '文档元数据表，代表一个被处理的源文件';
COMMENT ON COLUMN documents.id IS '主键ID';
COMMENT ON COLUMN documents.created_at IS '创建时间';
COMMENT ON COLUMN documents.updated_at IS '更新时间';
COMMENT ON COLUMN documents.source_uri IS '原始文件的来源路径';
COMMENT ON COLUMN documents.file_name IS '原始文件名';
COMMENT ON COLUMN documents.file_type IS '文件类型';
COMMENT ON COLUMN documents.file_size IS '文件大小(字节)';
COMMENT ON COLUMN documents.status IS '当前文档的处理状态';
COMMENT ON COLUMN documents.metadata_json IS '非结构化的元数据';


-- 3. 父文档内容块表 (document_parent_chunks)
CREATE TABLE document_parent_chunks (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    doc_id BIGINT, -- 关联到 documents 表，表明此内容块属于哪个文档
    content TEXT, -- 该逻辑块的纯文本内容
    chunk_order INT, -- 该父内容块在整个文档中的顺序，从0开始
    metadata_json TEXT -- 存储与此内容块相关的元数据，如章节标题、页码范围等
);

COMMENT ON TABLE document_parent_chunks IS '父文档内容表，存储从原始文档中按逻辑结构（如章节）划分出的大块内容';
COMMENT ON COLUMN document_parent_chunks.id IS '主键ID';
COMMENT ON COLUMN document_parent_chunks.created_at IS '创建时间';
COMMENT ON COLUMN document_parent_chunks.updated_at IS '更新时间';
COMMENT ON COLUMN document_parent_chunks.doc_id IS '关联的文档ID';
COMMENT ON COLUMN document_parent_chunks.content IS '该逻辑块的纯文本内容';
COMMENT ON COLUMN document_parent_chunks.chunk_order IS '该父内容块在整个文档中的顺序';
COMMENT ON COLUMN document_parent_chunks.metadata_json IS '相关的元数据';


-- 4. 子文档检索单元表 (document_chunks)
CREATE TABLE document_chunks (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    doc_id BIGINT, -- 冗余存储，直接关联到源文档
    parent_chunk_id BIGINT, -- 关联到其来源的父内容块
    chunk_content TEXT, -- 子文档块的实际文本内容
    token_count INT, -- chunk_content 的 token 数量
    embedding vector(1024), -- 内容的向量嵌入
    metadata_json TEXT, -- 存储与此子文档块相关的特定元数据
    start_position INT, -- 在父块内容中的起始字符位置
    end_position INT -- 在父块内容中的结束字符位置
);

COMMENT ON TABLE document_chunks IS '子文档检索单元表，存储被精细切分、用于向量化和最终检索的子文档';
COMMENT ON COLUMN document_chunks.id IS '主键ID';
COMMENT ON COLUMN document_chunks.created_at IS '创建时间';
COMMENT ON COLUMN document_chunks.updated_at IS '更新时间';
COMMENT ON COLUMN document_chunks.doc_id IS '关联的源文档ID';
COMMENT ON COLUMN document_chunks.parent_chunk_id IS '关联的父内容块ID';
COMMENT ON COLUMN document_chunks.chunk_content IS '子文档块的实际文本内容';
COMMENT ON COLUMN document_chunks.token_count IS 'token 数量';
COMMENT ON COLUMN document_chunks.embedding IS '内容的向量嵌入';
COMMENT ON COLUMN document_chunks.metadata_json IS '相关的特定元数据';
COMMENT ON COLUMN document_chunks.start_position IS '在父块内容中的起始字符位置';
COMMENT ON COLUMN document_chunks.end_position IS '在父块内容中的结束字符位置';

-- 为 embedding 字段创建 HNSW 索引
-- m 和 ef_construction 的值可以根据您的数据量和查询需求进行调整
CREATE INDEX ON document_chunks USING hnsw (embedding vector_l2_ops) WITH (m = 16, ef_construction = 64);


-- 5. 人物表 (person)
CREATE TABLE person (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    name VARCHAR(255) UNIQUE, -- 人物姓名，唯一索引
    details TEXT, -- 人物详细信息
    has_relationships BOOLEAN DEFAULT FALSE -- 是否已生成关系，默认为false
);

COMMENT ON TABLE person IS '人物表';
COMMENT ON COLUMN person.id IS '主键ID';
COMMENT ON COLUMN person.created_at IS '创建时间';
COMMENT ON COLUMN person.updated_at IS '更新时间';
COMMENT ON COLUMN person.name IS '人物姓名';
COMMENT ON COLUMN person.details IS '人物详细信息';
COMMENT ON COLUMN person.has_relationships IS '是否已生成关系';


-- 6. 历史事件表 (historical_events)
CREATE TABLE historical_events (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    name VARCHAR(255) UNIQUE, -- 历史事件名称，唯一
    summary TEXT -- 历史事件摘要
);

COMMENT ON TABLE historical_events IS '历史事件表';
COMMENT ON COLUMN historical_events.id IS '主键ID';
COMMENT ON COLUMN historical_events.created_at IS '创建时间';
COMMENT ON COLUMN historical_events.updated_at IS '更新时间';
COMMENT ON COLUMN historical_events.name IS '历史事件名称';
COMMENT ON COLUMN historical_events.summary IS '历史事件摘要';


-- 7. 时间轴轨道表 (timeline_tracks)
CREATE TABLE timeline_tracks (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    event_id BIGINT, -- 关联的历史事件ID
    name VARCHAR(255) -- 轨道名称
);

COMMENT ON TABLE timeline_tracks IS '时间轴轨道表';
COMMENT ON COLUMN timeline_tracks.id IS '主键ID';
COMMENT ON COLUMN timeline_tracks.created_at IS '创建时间';
COMMENT ON COLUMN timeline_tracks.updated_at IS '更新时间';
COMMENT ON COLUMN timeline_tracks.event_id IS '关联的历史事件ID';
COMMENT ON COLUMN timeline_tracks.name IS '轨道名称';


-- 8. 子事件节点表 (timeline_sub_events)
CREATE TABLE timeline_sub_events (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    track_id BIGINT, -- 关联的轨道ID
    event_date VARCHAR(255), -- 事件日期(文本形式)
    title VARCHAR(255), -- 事件标题
    description TEXT -- 事件详细描述
);

COMMENT ON TABLE timeline_sub_events IS '子事件节点表';
COMMENT ON COLUMN timeline_sub_events.id IS '主键ID';
COMMENT ON COLUMN timeline_sub_events.created_at IS '创建时间';
COMMENT ON COLUMN timeline_sub_events.updated_at IS '更新时间';
COMMENT ON COLUMN timeline_sub_events.track_id IS '关联的轨道ID';
COMMENT ON COLUMN timeline_sub_events.event_date IS '事件日期(文本形式)';
COMMENT ON COLUMN timeline_sub_events.title IS '事件标题';
COMMENT ON COLUMN timeline_sub_events.description IS '事件详细描述';


-- 9. 剧本表 (scenarios)
CREATE TABLE scenarios (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    title VARCHAR(255), -- 剧本标题
    description TEXT, -- 剧本的简要描述
    historical_background TEXT, -- 详细的历史背景和世界观设定
    gameplay_setting TEXT, -- 剧本玩法，剧本结局，剧本预设事件等设定
    style_setting TEXT, -- 剧本文风设定
    role TEXT, -- 剧本中预设的角色卡
    initial_prompt TEXT, -- 剧本开始的初始场景设定
    rag_document_ids JSONB -- 关联的史料文档ID列表
);

COMMENT ON TABLE scenarios IS '剧本表';
COMMENT ON COLUMN scenarios.id IS '主键ID';
COMMENT ON COLUMN scenarios.created_at IS '创建时间';
COMMENT ON COLUMN scenarios.updated_at IS '更新时间';
COMMENT ON COLUMN scenarios.title IS '剧本标题';
COMMENT ON COLUMN scenarios.description IS '剧本的简要描述';
COMMENT ON COLUMN scenarios.historical_background IS '详细的历史背景和世界观设定';
COMMENT ON COLUMN scenarios.gameplay_setting IS '剧本玩法设定';
COMMENT ON COLUMN scenarios.style_setting IS '剧本文风设定';
COMMENT ON COLUMN scenarios.role IS '剧本中预设的角色卡';
COMMENT ON COLUMN scenarios.initial_prompt IS '剧本开始的初始场景设定';
COMMENT ON COLUMN scenarios.rag_document_ids IS '关联的史料文档ID列表';


-- 10. 游戏会话表 (game_sessions)
CREATE TABLE game_sessions (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    user_id VARCHAR(255), -- 用户ID
    scenario_id BIGINT -- 关联的剧本ID
);

COMMENT ON TABLE game_sessions IS '游戏会话表，记录用户与游戏会话的关联关系';
COMMENT ON COLUMN game_sessions.id IS '主键ID';
COMMENT ON COLUMN game_sessions.created_at IS '创建时间';
COMMENT ON COLUMN game_sessions.updated_at IS '更新时间';
COMMENT ON COLUMN game_sessions.user_id IS '用户ID';
COMMENT ON COLUMN game_sessions.scenario_id IS '关联的剧本ID';


-- 11. 文档分析会话表 (analysis_sessions)
CREATE TABLE analysis_sessions (
    id BIGSERIAL PRIMARY KEY, -- 主键ID
    created_at TIMESTAMP, -- 创建时间
    updated_at TIMESTAMP, -- 更新时间
    user_id BIGINT, -- 用户ID
    document_ids JSONB -- 关联的文档ID列表 (以JSON格式存储)
);

COMMENT ON TABLE analysis_sessions IS '文档分析会话表';
COMMENT ON COLUMN analysis_sessions.id IS '主键ID';
COMMENT ON COLUMN analysis_sessions.created_at IS '创建时间';
COMMENT ON COLUMN analysis_sessions.updated_at IS '更新时间';
COMMENT ON COLUMN analysis_sessions.user_id IS '用户ID';
COMMENT ON COLUMN analysis_sessions.document_ids IS '关联的文档ID列表';