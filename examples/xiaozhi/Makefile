.PHONY: help install install-rust install-opus install-onnxruntime install-models install-pyaudio install-tkinter build dev dev-gui dev-xiaozhi stop clean test lint format docker-build docker-run venv

# 虚拟环境路径
VENV := .venv
PYTHON := $(VENV)/bin/python
UV := uv
ONNX_VERSION := 1.23.2

# 检测系统架构
UNAME_M := $(shell uname -m)
ifeq ($(UNAME_M),arm64)
    ORT_ARCH := osx-arm64
else ifeq ($(UNAME_M),x86_64)
    ORT_ARCH := osx-x86_64
else
    $(error 不支持的架构: $(UNAME_M))
endif

# 默认目标
help:
	@echo "可用命令："
	@echo ""
	@echo "安装依赖："
	@echo "  make install             - 安装所有依赖（Python + Rust + Opus + ONNX Runtime + 模型）"
	@echo "  make install-rust        - 安装 Rust 工具链"
	@echo "  make install-opus        - 安装 Opus 音频编解码库"
	@echo "  make install-onnxruntime - 下载并配置 ONNX Runtime 动态库"
	@echo "  make install-models      - 下载 VAD + KWS 模型文件"
	@echo "  make install-pyaudio     - 安装 PyAudio（xiaozhi 模式需要）"
	@echo "  make install-tkinter     - 安装 tkinter（GUI 模式需要）"
	@echo "  make venv                - 创建 Python 虚拟环境"
	@echo "  make build               - 编译 Rust 扩展模块"
	@echo ""
	@echo "启动服务："
	@echo "  make dev                 - 启动服务（CLI 模式，连接小爱音箱）⭐️ 推荐"
	@echo "  make dev-xiaozhi         - 启动服务（小智 AI 模式，使用电脑麦克风，需要 PyAudio）"
	@echo "  make dev-gui             - 启动服务（GUI 模式，需要 tkinter）"
	@echo "  make stop                - 停止正在运行的服务"
	@echo ""
	@echo "其他命令："
	@echo "  make clean               - 清理构建文件"
	@echo "  make docker-build        - 构建 Docker 镜像"
	@echo "  make docker-run          - 运行 Docker 容器"

# 检查 uv 是否安装
check-uv:
	@command -v uv >/dev/null 2>&1 || { echo "❌ 错误: uv 未安装，请访问 https://github.com/astral-sh/uv 安装"; exit 1; }
	@echo "✅ uv 已安装"

# 检查 Rust 是否安装
check-rust:
	@command -v cargo >/dev/null 2>&1 || { echo "❌ 错误: Rust 未安装，请访问 https://www.rust-lang.org/learn/get-started 安装"; exit 1; }
	@echo "✅ Rust 已安装"

# 安装 Rust（如果未安装）
install-rust:
	@if ! command -v cargo >/dev/null 2>&1; then \
		echo "📦 正在安装 Rust..."; \
		curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y; \
		echo "✅ Rust 安装完成"; \
	else \
		echo "✅ Rust 已安装"; \
	fi

# 安装 Opus（macOS）
install-opus:
	@if [ "$$(uname)" = "Darwin" ]; then \
		if ! command -v brew >/dev/null 2>&1; then \
			echo "❌ 错误: Homebrew 未安装，请先安装 Homebrew"; \
			exit 1; \
		fi; \
		if ! brew list opus >/dev/null 2>&1; then \
			echo "📦 正在安装 Opus..."; \
			brew install opus; \
			echo "✅ Opus 安装完成"; \
		else \
			echo "✅ Opus 已安装"; \
		fi; \
	else \
		echo "⚠️  请手动安装 Opus 库（Linux: apt-get install libopus-dev）"; \
	fi

# 创建虚拟环境
venv: check-uv
	@if [ ! -d "$(VENV)" ]; then \
		echo "📦 创建 Python 虚拟环境..."; \
		$(UV) venv; \
		echo "✅ 虚拟环境创建完成"; \
	else \
		echo "✅ 虚拟环境已存在"; \
	fi

# 安装 Python 依赖
install-python: check-uv venv
	@echo "📦 正在安装 Python 依赖..."
	$(UV) sync --locked
	@echo "✅ Python 依赖安装完成"

# 编译 Rust 扩展
build: check-rust venv
	@echo "🔨 正在编译 Rust 扩展模块..."
	. $(VENV)/bin/activate && $(UV) run maturin build --release
	@echo "✅ 编译完成"

# 下载并配置 ONNX Runtime
install-onnxruntime: venv
	@if [ ! -f "$(VENV)/lib/python3.12/site-packages/onnxruntime/capi/libonnxruntime.$(ONNX_VERSION).dylib" ]; then \
		echo "📦 正在下载 ONNX Runtime $(ONNX_VERSION) ($(ORT_ARCH))..."; \
		mkdir -p /tmp/onnxruntime; \
		curl -fsSL "https://github.com/microsoft/onnxruntime/releases/download/v$(ONNX_VERSION)/onnxruntime-$(ORT_ARCH)-$(ONNX_VERSION).tgz" -o /tmp/onnxruntime.tgz; \
		tar -xzf /tmp/onnxruntime.tgz -C /tmp/onnxruntime; \
		mkdir -p $(VENV)/lib/python3.12/site-packages/onnxruntime/capi; \
		cp /tmp/onnxruntime/onnxruntime-$(ORT_ARCH)-$(ONNX_VERSION)/lib/libonnxruntime.$(ONNX_VERSION).dylib $(VENV)/lib/python3.12/site-packages/onnxruntime/capi/; \
		cd $(VENV)/lib/python3.12/site-packages/onnxruntime/capi && ln -sf libonnxruntime.$(ONNX_VERSION).dylib libonnxruntime.dylib; \
		rm -rf /tmp/onnxruntime /tmp/onnxruntime.tgz; \
		echo "✅ ONNX Runtime 安装完成"; \
	else \
		echo "✅ ONNX Runtime 已安装"; \
	fi

# 下载模型文件
install-models:
	@if [ ! -f "xiaozhi/models/silero_vad.onnx" ]; then \
		echo "📦 正在下载 VAD + KWS 模型文件（约 103MB）..."; \
		curl -L "https://github.com/idootop/open-xiaoai/releases/download/vad-kws-models/models.zip" -o /tmp/models.zip; \
		unzip -q /tmp/models.zip -d xiaozhi/; \
		rm -f /tmp/models.zip; \
		echo "✅ 模型文件下载完成"; \
	else \
		echo "✅ 模型文件已存在"; \
	fi

# 安装所有依赖
install: install-rust install-opus install-python install-onnxruntime install-models
	@echo "✅ 所有依赖安装完成"

# 安装 PyAudio（xiaozhi 模式需要）
install-pyaudio: venv
	@echo "📦 安装 PyAudio..."
	@if [ "$$(uname)" = "Darwin" ]; then \
		if ! brew list portaudio >/dev/null 2>&1; then \
			echo "正在安装 portaudio..."; \
			brew install portaudio; \
		fi; \
	fi
	. $(VENV)/bin/activate && $(UV) pip install pyaudio
	@echo "✅ PyAudio 安装完成"

# 安装 tkinter（GUI 模式需要）
install-tkinter:
	@echo "📦 安装 tkinter..."
	@if [ "$$(uname)" = "Darwin" ]; then \
		echo "在 macOS 上，需要重新安装带 tkinter 支持的 Python"; \
		echo "使用 pyenv 的话，运行："; \
		echo "  env PYTHON_CONFIGURE_OPTS='--with-tcltk-includes=\"-I$$(brew --prefix tcl-tk)/include\" --with-tcltk-libs=\"-L$$(brew --prefix tcl-tk)/lib -ltcl8.6 -ltk8.6\"' pyenv install 3.12.2"; \
		echo "或者使用 CLI 模式：make dev"; \
	else \
		echo "在 Linux 上，运行："; \
		echo "  sudo apt-get install python3-tk"; \
	fi

# 启动服务（CLI 模式，支持唤醒词）⭐️ 推荐
dev: venv
	@echo "🚀 启动服务（CLI 模式）..."
	. $(VENV)/bin/activate && \
	export DYLD_LIBRARY_PATH=$(VENV)/lib/python3.12/site-packages/onnxruntime/capi:$$DYLD_LIBRARY_PATH && \
	CLI=true $(PYTHON) main.py

# 启动服务（GUI 模式，需要 tkinter）
dev-gui: venv
	@echo "🚀 启动服务（GUI 模式）..."
	@echo "⚠️  如果遇到 tkinter 错误，请运行: make install-tkinter"
	. $(VENV)/bin/activate && \
	export DYLD_LIBRARY_PATH=$(VENV)/lib/python3.12/site-packages/onnxruntime/capi:$$DYLD_LIBRARY_PATH && \
	$(PYTHON) main.py

# 启动服务（小智 AI 模式，需要 PyAudio）
dev-xiaozhi: venv
	@echo "🚀 启动服务（小智 AI 模式）..."
	@echo "⚠️  如果遇到 PyAudio 错误，请运行: make install-pyaudio"
	. $(VENV)/bin/activate && \
	export DYLD_LIBRARY_PATH=$(VENV)/lib/python3.12/site-packages/onnxruntime/capi:$$DYLD_LIBRARY_PATH && \
	CLI=true $(PYTHON) main.py --mode xiaozhi

# 清理构建文件
clean:
	@echo "🧹 清理构建文件..."
	rm -rf target/
	rm -rf .venv/
	rm -rf *.egg-info/
	rm -rf dist/
	rm -rf build/
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
	find . -type f -name "*.pyc" -delete 2>/dev/null || true
	@echo "✅ 清理完成"

# 构建 Docker 镜像
docker-build:
	@echo "🐳 构建 Docker 镜像..."
	docker build -t open-xiaoai-xiaozhi:latest -f Dockerfile ../..
	@echo "✅ Docker 镜像构建完成"

# 运行 Docker 容器
docker-run:
	@echo "🐳 运行 Docker 容器..."
	docker run -it --rm -p 4399:4399 -v $$(pwd)/config.py:/app/config.py open-xiaoai-xiaozhi:latest

# 格式化代码
format: venv
	@echo "🎨 格式化 Python 代码..."
	. $(VENV)/bin/activate && $(UV) run ruff format . || echo "⚠️  ruff 未安装，跳过格式化"

# 代码检查
lint: venv
	@echo "🔍 检查代码..."
	. $(VENV)/bin/activate && $(UV) run ruff check . || echo "⚠️  ruff 未安装，跳过检查"

# 运行测试
test: venv
	@echo "🧪 运行测试..."
	. $(VENV)/bin/activate && $(UV) run pytest || echo "⚠️  pytest 未安装，跳过测试"

# 停止服务
stop:
	@echo "🛑 停止服务..."
	@pkill -f "python main.py" || echo "没有运行中的服务"
	@echo "✅ 服务已停止"
