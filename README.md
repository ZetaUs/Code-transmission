# Code

## Windows box
```
import os
import subprocess

def create_and_launch_sandbox():
    # 1. 定义 .wsb 配置文件的内容（XML 格式）
    # 你可以在这里像搭积木一样，定制你的专属沙盒环境
    wsb_content = """<Configuration>
  <!-- 启用虚拟GPU，让沙盒里的界面更流畅 -->
  <VGpu>Enable</VGpu>
  
  <!-- 禁用网络！打造一个绝对断网的单机测试环境 -->
  <Networking>Disable</Networking>
  
  <!-- 映射主机文件夹：把你电脑上的文件夹“刻录”进沙盒 -->
  <MappedFolders>
    <MappedFolder>
      <!-- 这里写你电脑上真实存在的文件夹路径 -->
      <HostFolder>C:\\Users\\Public\\Documents</HostFolder>
      <!-- 设为 true 表示沙盒里只能看不能改，保护你的原文件 -->
      <ReadOnly>true</ReadOnly>
    </MappedFolder>
  </MappedFolders>

  <!-- 登录命令：沙盒启动后，自动帮你打开记事本 -->
  <LogonCommand>
    <Command>notepad.exe</Command>
  </LogonCommand>
</Configuration>
"""

    # 2. 将配置内容“刻录”成一个真实的 .wsb 文件
    wsb_filename = "My_Custom_Sandbox.wsb"
    print(f"--- 🛠️ 正在刻录专属沙盒配置文件: {wsb_filename} ---")
    with open(wsb_filename, "w", encoding="utf-8") as f:
        f.write(wsb_content)
    
    # 3. 指挥 Windows 启动这个定制好的沙盒
    print("--- 🚀 正在启动你的专属 Windows 沙盒... ---")
    try:
        # 使用 os.startfile 可以直接双击打开 .wsb 文件
        os.startfile(wsb_filename)
        print("✅ 专属沙盒已成功弹出！你可以去里面随意折腾了。")
    except Exception as e:
        print(f"⛔ 启动失败，请检查你的电脑是否已开启 Windows 沙盒功能。\n报错: {e}")

if __name__ == "__main__":
    create_and_launch_sandbox()
```

## code_anim.py
```
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
import pygame
import sys
import numpy as np

# ---------------------- 1. 数据与模型部分 ----------------------
# 构建文本数据
texts = [
    "hello world",
    "hello yellow",
    "coding is fun",
    "python is cool",
    "animation vs code"
]

# 构建词汇表
vocab = set()
for text in texts:
    vocab.update(text.split())
vocab = sorted(vocab)
word_to_idx = {word: idx for idx, word in enumerate(vocab)}
idx_to_word = {idx: word for word, idx in word_to_idx.items()}
vocab_size = len(vocab)

seq_length = 2  # 用前2个词预测第3个

# 文本数据集类
class TextDataset(Dataset):
    def __init__(self, texts, word_to_idx, seq_length=2):
        self.texts = texts
        self.word_to_idx = word_to_idx
        self.seq_length = seq_length
        self.data = []
        for text in texts:
            tokens = text.split()
            # 生成 (前n个词, 下一个词)
            for i in range(len(tokens) - seq_length):
                x = tokens[i:i+seq_length]
                y = tokens[i+seq_length]
                self.data.append((x, y))
    
    def __len__(self):
        return len(self.data)
    
    def __getitem__(self, idx):
        x_tokens, y_token = self.data[idx]
        x = torch.tensor([self.word_to_idx[tok] for tok in x_tokens], dtype=torch.long)
        y = torch.tensor(self.word_to_idx[y_token], dtype=torch.long)
        return x, y

# LSTM 模型
class LSTMModel(nn.Module):
    def __init__(self, vocab_size, embedding_dim=16, hidden_dim=32, seq_length=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_dim, batch_first=True)
        self.fc = nn.Linear(hidden_dim, vocab_size)
    
    def forward(self, x):
        embedded = self.embedding(x)
        lstm_out, _ = self.lstm(embedded)
        out = self.fc(lstm_out[:, -1, :])  # 取最后一步输出
        return out

# 初始化
dataset = TextDataset(texts, word_to_idx, seq_length)
dataloader = DataLoader(dataset, batch_size=2, shuffle=True)
model = LSTMModel(vocab_size)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.01)

# 训练
epochs = 100
for epoch in range(epochs):
    total_loss = 0
    for x, y in dataloader:
        optimizer.zero_grad()
        output = model(x)
        loss = criterion(output, y)
        loss.backward()
        optimizer.step()
        total_loss += loss.item()
    if (epoch + 1) % 10 == 0:
        avg_loss = total_loss / len(dataloader)
        print(f"Epoch {epoch+1:3d} | Loss: {avg_loss:.4f}")

# 预测函数（自动处理长度）
def predict_next_word(input_sequence, model, word_to_idx, idx_to_word, seq_len=2):
    tokens = input_sequence.strip().split()
    # 只取最后 seq_len 个词
    tokens = tokens[-seq_len:]
    # 如果不够，直接返回空
    if len(tokens) != seq_len:
        return "need more words"
    
    try:
        x = torch.tensor([[word_to_idx[w] for w in tokens]], dtype=torch.long)
        with torch.no_grad():
            out = model(x)
            idx = torch.argmax(out).item()
        return idx_to_word[idx]
    except KeyError as e:
        return f"unknown word: {str(e)}"

# ---------------------- 2. Pygame 可视化交互 ----------------------
pygame.init()
W, H = 800, 600
screen = pygame.display.set_mode((W, H))
pygame.display.set_caption("Animation vs Code - LSTM Text Predictor")
font = pygame.font.SysFont("Consolas", 26)
clock = pygame.time.Clock()

input_text = ""
predicted_word = ""

while True:
    screen.fill((0, 0, 0))
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
        
        elif event.type == pygame.KEYDOWN:
            if event.key == pygame.K_RETURN:
                # 回车预测
                predicted_word = predict_next_word(input_text, model, word_to_idx, idx_to_word, seq_length)
            elif event.key == pygame.K_BACKSPACE:
                input_text = input_text[:-1]
            elif event.key == pygame.K_ESCAPE:
                input_text = ""
                predicted_word = ""
            else:
                # 只允许可见字符
                if event.unicode.isprintable():
                    input_text += event.unicode

    # 绘制
    surf_input = font.render(f"Input: {input_text}", True, (220, 220, 220))
    screen.blit(surf_input, (50, 100))

    if predicted_word:
        color = (0, 255, 100) if "unknown" not in predicted_word and "need" not in predicted_word else (255, 80, 80)
        surf_pred = font.render(f"Predicted: {predicted_word}", True, color)
        screen.blit(surf_pred, (50, 160))

    pygame.display.flip()
    clock.tick(60)
```
### 依赖
```
pip install torch pygame numpy
```
