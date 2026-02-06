# 1. Initialize LFS
git lfs install

# 2. Track the heavy weights and checkpoints
git lfs track "*.safetensors"
git lfs track "*.bin"
git lfs track "*.pt"
git lfs track "*.pth"
git lfs track "*.model"

# 3. Track large tokenizer files (Optional but recommended)
git lfs track "tokenizer.json"
git lfs track "tokenizer.model"

# 4. Make sure git sees the new tracking rules
git add .gitattributes