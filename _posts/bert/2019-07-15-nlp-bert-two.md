---
title: "nlp - bert(二)"
subtitle: "bert 预训练"
layout: post
author: "Wentao Dong"
date: 2019-07-15 21:00:00
catalog: false
header-style: post
header-img: "img/city_night.png"
tags:
  - NLP
  - BERT
---

1. wordpiece:
   1. Google 创建了一个wordpiece 的词典: [wordpiece 介绍](https://www.cnblogs.com/huangyc/p/10223075.html)
2. 创建预训练需要的数据：
   1. 分词创建document：
      1. convert_to_unicode：转成unicode字符
      2. clean_text: 移除控制字符、空字符(注意不是空格)、替换字符(一个占位符)；将空格、\t、\n、\r等 转化为空格
      3. tokenize_chinese: 中文字符前后加空格
      4. whitespace_tokenize : 按空格切词，空格不是词
      5. 根据参数配置执行是否转小写
      6. 移除变音符号
      7. split_on_punc: 按标点分词，标点自成一个词
      8. whitespace_tokenize：再次按空格分词
      9. wordpiece:
         1. 如果token长度超过max_input_chars_per_word，则添加为"[UNK]"，意为unknown
         2. 如果wordpiece 分词结果没有在词典中找到，则添加为"[UNK]"
   2. create_instances_from_document:
      1. 设置一个document的最大包含token 数量 for [CLS], [SEP], [SEP]：max_num_tokens = max_seq_length - 3
      2. 随机选择10%的document，设置target_seq_length 为一个较小的值，范围(2, max_num_tokens)
      3. 构造预训练数据
         1. next_sentence_task: 
            1. 选取current_chunk：当选取到document最后一个seg或者选取的seg总长大于等于target_seq_length 才进行构造训练数据
            2. 选取current_chunk中前部若干seg为tokens_a
            3. 如果current_chunk多于一句话, 随机取若干seg为tokens_a
               1. 50%的概率使用当前块的剩余部分作为tokens_b
               2. 50%的概率随机选择一个非当前document的部分连续seg作为tokens_b
            4. 如果current_chunk只有一句
               1. 概率随机选择一个非当前document的部分连续seg作为tokens_b
            5. 防止浪费数据，选非当前document的部分连续seg作为tokens_b，则 i-=num_unused_segments，因为没用真实的下一句，所以放回去
            6. truncate_seq_pair：裁剪策略为，选择tokens_a和tokens_b中较长的一个进行裁剪，50%的概率从前面删除，50%的概率从后面删除
         2. 构造tokens、segment_ids：
            1. tokens:           [CLS], x, x, x, x, [SEP], y, y, y, y, [SEP]
            2. segment_ids:     0,   0, 0, 0, 0,    0,    1, 1, 1, 1,    1
         3. create_masked_lm_predictions:
            1. 每个句子随机选择最多遮盖20 个或者15%的token，哪个数字小选哪个
               1. 80%的概率替换为[MASK]
               2. 10%的概率保持原token
               3. 10%的概率替换为词典中的随机一个token
            2. 返回一个含有遮盖token的 tokens列表、遮盖位置masked_lm_positions列表、被遮盖的词masked_lm_labels 列表
         4. TrainingInstance：tokens、segment_ids、is_random_next、masked_lm_positions、masked_lm_labels
   3. write_instance_to_example_files：
      1. 根据tokens构造input_ids、input_mask
         1. input_ids是token在词典中的位置
         2. input_mask是告诉模型哪些是训练数据，哪些是padding
      2. 根据masked_lm_labels构造masked_lm_ids、masked_lm_weights
         1. masked_lm_ids是被遮盖的词在词典中的位置
         2. masked_lm_weights是被遮盖词的权重
      3. 根据next_sentence_label=is_random_next? 1:0
      4. 构造features：input_ids、input_mask、segment_ids、masked_lm_positions、masked_lm_ids、masked_lm_weights、next_sentence_labels
3. 预训练：
   1. 构造BertModel
   2. get_masked_lm_output
      1. 取出BertModel最后一层输出作为input_tensor
      2. 根据masked_lm_positions从input_tensor中取出被遮盖词的tensor，展平(8,20,768)->(160, 768)
      3. 加一层全链接 & layer_norm
      4. 加一层softmax链接，得到每个位置到词典中每个词的概率
      5. 根据masked_lm_weights计算交叉熵损失masked_lm_loss
   3. get_next_sentence_output
      1. 取出BertModel最后一层第一个位置的输出作为input_tensor
      2. 加一层softmax链接，得到is_next_sentence的概率
      3. 计算交叉熵损失next_sentence_loss
   4. 总损失total_loss = masked_lm_loss + next_sentence_loss