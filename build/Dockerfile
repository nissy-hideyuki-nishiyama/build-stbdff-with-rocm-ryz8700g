# ベースイメージ: ROCm 7.1とPyTorchがプリインストールされた公式イメージ
# ※ rocm/pytorch:latest は 7.x系を指していることが多いですが、正確なタグを推奨
FROM rocm/pytorch:rocm7.1_ubuntu24.04_py3.12_pytorch_release_2.8.0

# コンテナ内の作業ディレクトリを設定
WORKDIR /app

# ComfyUIのクローンと初期セットアップ
RUN git clone https://github.com/comfyanonymous/ComfyUI.git /app/ComfyUI
WORKDIR /app/ComfyUI

# 必要な依存関係をインストール
# PyTorch/Torchvisionはベースイメージにあるため除外。それ以外のStable Diffusion/ComfyUIに必要なものをインストール。
# requirements_gpu.txt を使用するか、requirements.txtからtorch関連を除外する
RUN grep -vE 'torch$|torchvision$|torchaudio$' requirements.txt > temp_requirements.txt \
    && pip install --no-cache-dir -r temp_requirements.txt \
    # && pip uninstall -y torch torchvision torchaudio \
    && pip install --no-cache-dir \
        huggingface-hub \
        diffusers \
        accelerate \
        transformers \
        safetensors \
        # ComfyUI Managerが必要な場合は追加 (任意)
        # git clone https://github.com/Comfy-Org/ComfyUI-Manager.git custom_nodes/ComfyUI-Manager

# エントリポイントの作成 (WebUI起動スクリプト)
# --listen 0.0.0.0 で外部からアクセス可能にする
# --port 8188 はデフォルトだが明示
ENTRYPOINT ["python", "/app/ComfyUI/main.py", "--listen", "0.0.0.0", "--port", "8188", "--lowvram", "--cpu-vae"]
