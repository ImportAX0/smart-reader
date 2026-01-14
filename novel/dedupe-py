import os
import re
from difflib import SequenceMatcher
from datetime import datetime

def extract_title_author_strict(filename):
    """更严格的标题和作者提取"""
    name = re.sub(r'\.txt$', '', filename, flags=re.IGNORECASE)
    original_name = name
    
    # 更精确的作者识别模式
    author = None
    author_patterns = [
        r'作者[：:][\s]*([^\s\[\]【】《》（）()\d]{2,10})',  # 作者:张三
        r'[\s\-_]by[\s]+([a-zA-Z\u4e00-\u9fa5]{2,15})(?:[\s\-_]|$)',  # by 张三
        r'【([^\[\]【】]{2,10})】(?:作品|著)',  # 【张三】作品
    ]
    
    for pattern in author_patterns:
        match = re.search(pattern, name, re.IGNORECASE)
        if match:
            author = match.group(1).strip()
            # 移除已识别的作者部分
            name = re.sub(pattern, '', name, flags=re.IGNORECASE)
            break
    
    if not author:
        author = 'unknown'
    
    # 清理书名
    # 去除常见的标签和前缀(但保留书名号内的内容)
    name = re.sub(r'^\[[\w\s]+\][\s]*', '', name)  # [标签]
    name = re.sub(r'^\([\w\s]+\)[\s]*', '', name)  # (标签)
    # 只去除非书名的【】标签,保留【虫族】这类设定标签
    name = re.sub(r'^【(?:主攻|总攻|GB|女攻|np|NP)[\w\s]*】[\s]*', '', name)
    
    # 去除章节、状态标记
    name = re.sub(r'[（(]?(?:全本|完结|连载中|更新至\d+章?)[)）]?', '', name, flags=re.IGNORECASE)
    name = re.sub(r'[第]\d+[章卷集话].*
    
    # 提取书名核心
    # 优先匹配书名号内的内容
    title_match = re.search(r'《([^《》]+)》', name)
    if not title_match:
        title_match = re.search(r'【([^【】]+)】', name)
    if not title_match:
        # 如果没有书名号,直接使用清理后的name
        title = name
    else:
        title = title_match.group(1)
    
    # 清理书名
    title = re.sub(r'更\d+', '', title)  # 移除"更XX"
    title = re.sub(r'[\s\-_\d]+
    
    return title.lower().strip(), author.lower().strip()


def calculate_title_similarity(title1, title2):
    """计算两个标题的相似度"""
    return SequenceMatcher(None, title1, title2).ratio()


def read_file_sample(filepath, sample_size=2000):
    """读取文件开头部分用于内容比对"""
    try:
        with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read(sample_size)
    except:
        try:
            with open(filepath, 'r', encoding='gbk', errors='ignore') as f:
                return f.read(sample_size)
        except:
            return ""


def calculate_content_similarity(file1, file2):
    """计算两个文件内容的相似度"""
    content1 = read_file_sample(file1)
    content2 = read_file_sample(file2)
    
    if not content1 or not content2:
        return 0.0
    
    return SequenceMatcher(None, content1, content2).ratio()


def find_and_delete_duplicates(folder, 
                               auto_delete=True,
                               similarity_threshold=0.8,
                               content_similarity_threshold=0.85,
                               size_ratio_threshold=0.6):
    """
    查找并删除重复文件
    
    参数:
        folder: 目标文件夹
        auto_delete: True=自动删除高置信度重复, False=全部预览
        similarity_threshold: 书名相似度阈值(0-1)
        content_similarity_threshold: 内容相似度阈值,低于此值需人工审核
        size_ratio_threshold: 大小比例阈值,高于此值需内容检查
    """
    files = [f for f in os.listdir(folder) if f.lower().endswith('.txt')]
    
    if not files:
        print("未找到txt文件")
        return
    
    print(f"{'='*60}")
    print(f"开始处理: {folder}")
    print(f"共找到 {len(files)} 个txt文件")
    print(f"模式: {'自动删除高置信度重复' if auto_delete else '预览模式'}")
    print(f"{'='*60}\n")
    
    # 分组逻辑
    groups = {}
    
    for filename in files:
        title, author = extract_title_author_strict(filename)
        path = os.path.join(folder, filename)
        size = os.path.getsize(path)
        mtime = os.path.getmtime(path)
        
        # 创建分组键 - 必须标题和作者都匹配
        if author != 'unknown':
            key = f"{title}|{author}"
        else:
            # 作者未知的文件,使用完整书名作为键,避免误归组
            # 加入文件名hash确保不同文件不会被归到一起
            key = f"{title}|_noauthor_{hash(filename) % 10000}"
        
        if key not in groups:
            groups[key] = []
        
        groups[key].append({
            'filename': filename,
            'title': title,
            'author': author,
            'path': path,
            'size': size,
            'mtime': mtime
        })
    
    # 统计
    auto_deleted = 0
    manual_review_groups = []
    skipped_groups = []
    
    # 处理每个分组
    for key, file_list in groups.items():
        if len(file_list) == 1:
            continue  # 单文件无需处理
        
        title, author = key.rsplit('|', 1)
        
        # 检查是否为作者未知的误归组
        if author.startswith('_noauthor'):
            skipped_groups.append((title, file_list))
            continue
        
        # 检查书名相似度(针对同作者的多个文件)
        if author != 'unknown':
            titles = [f['title'] for f in file_list]
            if len(titles) > 1:
                # 计算所有配对的平均相似度
                similarities = []
                for i in range(len(titles)):
                    for j in range(i+1, len(titles)):
                        sim = calculate_title_similarity(titles[i], titles[j])
                        similarities.append(sim)
                
                avg_similarity = sum(similarities) / len(similarities) if similarities else 1.0
                
                if avg_similarity < similarity_threshold:
                    # 书名差异过大,可能是不同的书
                    manual_review_groups.append({
                        'reason': f'书名相似度过低({avg_similarity:.2f})',
                        'title': title,
                        'author': author,
                        'files': file_list
                    })
                    continue
        
        # 按大小和修改时间排序
        file_list.sort(key=lambda x: (-x['size'], -x['mtime']))
        main_file = file_list[0]
        
        # 检查其他文件
        has_suspicious = False
        files_to_delete = []
        
        for dup_file in file_list[1:]:
            size_ratio = dup_file['size'] / max(main_file['size'], 1)
            
            # 检查内容相似度
            should_check_content = size_ratio > size_ratio_threshold
            content_sim = -1
            
            if should_check_content:
                content_sim = calculate_content_similarity(main_file['path'], dup_file['path'])
                
                # 内容相似度过低,需要人工审核
                if content_sim < content_similarity_threshold:
                    has_suspicious = True
                    manual_review_groups.append({
                        'reason': f'内容差异较大(相似度={content_sim:.2f})',
                        'title': title,
                        'author': author,
                        'files': [main_file, dup_file]
                    })
                    continue
            
            # 可以安全删除
            files_to_delete.append({
                'file': dup_file,
                'size_ratio': size_ratio,
                'content_sim': content_sim
            })
        
        # 执行删除或预览
        if files_to_delete:
            if auto_delete and not has_suspicious:
                # 自动删除
                print(f"✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    try:
                        os.remove(dup_file['path'])
                        auto_deleted += 1
                        print(f"  ✗ 已删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    except Exception as e:
                        print(f"  ✗ 删除失败: {dup_file['filename']} - {e}")
            else:
                # 预览模式或有可疑情况
                print(f"\n{'='*60}")
                print(f"分组: 书名='{title}' | 作者='{author}'")
                print(f"  ✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    print(f"  ✗ 将删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    if item['content_sim'] >= 0:
                        print(f"     内容相似度: {item['content_sim']:.2f}")
    
    # 输出需要人工审核的分组
    if manual_review_groups:
        print(f"\n{'='*60}")
        print(f"⚠️  需要人工审核的分组 ({len(manual_review_groups)}个):")
        print(f"{'='*60}")
        
        for idx, group in enumerate(manual_review_groups, 1):
            print(f"\n[{idx}] {group['reason']}")
            print(f"    书名: {group['title']}")
            print(f"    作者: {group['author']}")
            for f in group['files']:
                print(f"      - {f['filename']} ({f['size']} bytes)")
    
    # 输出跳过的分组(作者未知且可能误归组)
    if skipped_groups:
        print(f"\n{'='*60}")
        print(f"ℹ️  跳过的分组 ({len(skipped_groups)}个) - 作者未知,已自动分离:")
        print(f"{'='*60}")
        
        for title, file_list in skipped_groups[:5]:  # 只显示前5个
            print(f"\n  书名: {title}")
            for f in file_list:
                print(f"    - {f['filename']}")
        
        if len(skipped_groups) > 5:
            print(f"\n  ... 还有 {len(skipped_groups)-5} 个分组被跳过")
    
    # 总结
    print(f"\n{'='*60}")
    print(f"处理完成!")
    print(f"{'实际' if auto_delete else '预计'}删除文件数: {auto_deleted}")
    print(f"需要人工审核的分组数: {len(manual_review_groups)}")
    print(f"跳过的分组数: {len(skipped_groups)}")
    print(f"{'='*60}\n")


# ========== 执行 ==========

# 直接执行删除,只预览可疑情况
find_and_delete_duplicates(
    folder=r"D:\脚本测试\txt",
    auto_delete=True,  # 自动删除高置信度重复
    similarity_threshold=0.8,  # 书名相似度阈值
    content_similarity_threshold=0.85,  # 内容相似度阈值(低于此值需人工审核)
    size_ratio_threshold=0.6  # 大小比例阈值(高于此值才检查内容)
), '', name)
    name = re.sub(r'[\s]*(?:txt|TXT)[\s]*
    
    # 提取书名核心（去除书名号）
    title_match = re.search(r'[《\[]?([^《》\[\]【】]{2,})[\]》]?', name)
    if title_match:
        title = title_match.group(1)
    else:
        title = name
    
    # 清理书名末尾
    title = re.sub(r'[\s\-_\d]+$', '', title)
    title = title.strip()
    
    # 如果书名太短或为空,使用原始文件名
    if not title or len(title) < 2:
        title = original_name
    
    # 移除书名中的"更XX"
    title = re.sub(r'更\d+', '', title).strip()
    
    return title.lower().strip(), author.lower().strip()


def calculate_title_similarity(title1, title2):
    """计算两个标题的相似度"""
    return SequenceMatcher(None, title1, title2).ratio()


def read_file_sample(filepath, sample_size=2000):
    """读取文件开头部分用于内容比对"""
    try:
        with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read(sample_size)
    except:
        try:
            with open(filepath, 'r', encoding='gbk', errors='ignore') as f:
                return f.read(sample_size)
        except:
            return ""


def calculate_content_similarity(file1, file2):
    """计算两个文件内容的相似度"""
    content1 = read_file_sample(file1)
    content2 = read_file_sample(file2)
    
    if not content1 or not content2:
        return 0.0
    
    return SequenceMatcher(None, content1, content2).ratio()


def find_and_delete_duplicates(folder, 
                               auto_delete=True,
                               similarity_threshold=0.8,
                               content_similarity_threshold=0.85,
                               size_ratio_threshold=0.6):
    """
    查找并删除重复文件
    
    参数:
        folder: 目标文件夹
        auto_delete: True=自动删除高置信度重复, False=全部预览
        similarity_threshold: 书名相似度阈值(0-1)
        content_similarity_threshold: 内容相似度阈值,低于此值需人工审核
        size_ratio_threshold: 大小比例阈值,高于此值需内容检查
    """
    files = [f for f in os.listdir(folder) if f.lower().endswith('.txt')]
    
    if not files:
        print("未找到txt文件")
        return
    
    print(f"{'='*60}")
    print(f"开始处理: {folder}")
    print(f"共找到 {len(files)} 个txt文件")
    print(f"模式: {'自动删除高置信度重复' if auto_delete else '预览模式'}")
    print(f"{'='*60}\n")
    
    # 分组逻辑
    groups = {}
    
    for filename in files:
        title, author = extract_title_author_strict(filename)
        path = os.path.join(folder, filename)
        size = os.path.getsize(path)
        mtime = os.path.getmtime(path)
        
        # 创建分组键 - 必须标题和作者都匹配
        if author != 'unknown':
            key = f"{title}|{author}"
        else:
            # 作者未知的文件,使用完整书名作为键,避免误归组
            # 加入文件名hash确保不同文件不会被归到一起
            key = f"{title}|_noauthor_{hash(filename) % 10000}"
        
        if key not in groups:
            groups[key] = []
        
        groups[key].append({
            'filename': filename,
            'title': title,
            'author': author,
            'path': path,
            'size': size,
            'mtime': mtime
        })
    
    # 统计
    auto_deleted = 0
    manual_review_groups = []
    skipped_groups = []
    
    # 处理每个分组
    for key, file_list in groups.items():
        if len(file_list) == 1:
            continue  # 单文件无需处理
        
        title, author = key.rsplit('|', 1)
        
        # 检查是否为作者未知的误归组
        if author.startswith('_noauthor'):
            skipped_groups.append((title, file_list))
            continue
        
        # 检查书名相似度(针对同作者的多个文件)
        if author != 'unknown':
            titles = [f['title'] for f in file_list]
            if len(titles) > 1:
                # 计算所有配对的平均相似度
                similarities = []
                for i in range(len(titles)):
                    for j in range(i+1, len(titles)):
                        sim = calculate_title_similarity(titles[i], titles[j])
                        similarities.append(sim)
                
                avg_similarity = sum(similarities) / len(similarities) if similarities else 1.0
                
                if avg_similarity < similarity_threshold:
                    # 书名差异过大,可能是不同的书
                    manual_review_groups.append({
                        'reason': f'书名相似度过低({avg_similarity:.2f})',
                        'title': title,
                        'author': author,
                        'files': file_list
                    })
                    continue
        
        # 按大小和修改时间排序
        file_list.sort(key=lambda x: (-x['size'], -x['mtime']))
        main_file = file_list[0]
        
        # 检查其他文件
        has_suspicious = False
        files_to_delete = []
        
        for dup_file in file_list[1:]:
            size_ratio = dup_file['size'] / max(main_file['size'], 1)
            
            # 检查内容相似度
            should_check_content = size_ratio > size_ratio_threshold
            content_sim = -1
            
            if should_check_content:
                content_sim = calculate_content_similarity(main_file['path'], dup_file['path'])
                
                # 内容相似度过低,需要人工审核
                if content_sim < content_similarity_threshold:
                    has_suspicious = True
                    manual_review_groups.append({
                        'reason': f'内容差异较大(相似度={content_sim:.2f})',
                        'title': title,
                        'author': author,
                        'files': [main_file, dup_file]
                    })
                    continue
            
            # 可以安全删除
            files_to_delete.append({
                'file': dup_file,
                'size_ratio': size_ratio,
                'content_sim': content_sim
            })
        
        # 执行删除或预览
        if files_to_delete:
            if auto_delete and not has_suspicious:
                # 自动删除
                print(f"✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    try:
                        os.remove(dup_file['path'])
                        auto_deleted += 1
                        print(f"  ✗ 已删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    except Exception as e:
                        print(f"  ✗ 删除失败: {dup_file['filename']} - {e}")
            else:
                # 预览模式或有可疑情况
                print(f"\n{'='*60}")
                print(f"分组: 书名='{title}' | 作者='{author}'")
                print(f"  ✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    print(f"  ✗ 将删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    if item['content_sim'] >= 0:
                        print(f"     内容相似度: {item['content_sim']:.2f}")
    
    # 输出需要人工审核的分组
    if manual_review_groups:
        print(f"\n{'='*60}")
        print(f"⚠️  需要人工审核的分组 ({len(manual_review_groups)}个):")
        print(f"{'='*60}")
        
        for idx, group in enumerate(manual_review_groups, 1):
            print(f"\n[{idx}] {group['reason']}")
            print(f"    书名: {group['title']}")
            print(f"    作者: {group['author']}")
            for f in group['files']:
                print(f"      - {f['filename']} ({f['size']} bytes)")
    
    # 输出跳过的分组(作者未知且可能误归组)
    if skipped_groups:
        print(f"\n{'='*60}")
        print(f"ℹ️  跳过的分组 ({len(skipped_groups)}个) - 作者未知,已自动分离:")
        print(f"{'='*60}")
        
        for title, file_list in skipped_groups[:5]:  # 只显示前5个
            print(f"\n  书名: {title}")
            for f in file_list:
                print(f"    - {f['filename']}")
        
        if len(skipped_groups) > 5:
            print(f"\n  ... 还有 {len(skipped_groups)-5} 个分组被跳过")
    
    # 总结
    print(f"\n{'='*60}")
    print(f"处理完成!")
    print(f"{'实际' if auto_delete else '预计'}删除文件数: {auto_deleted}")
    print(f"需要人工审核的分组数: {len(manual_review_groups)}")
    print(f"跳过的分组数: {len(skipped_groups)}")
    print(f"{'='*60}\n")


# ========== 执行 ==========

# 直接执行删除,只预览可疑情况
find_and_delete_duplicates(
    folder=r"D:\脚本测试\txt",
    auto_delete=True,  # 自动删除高置信度重复
    similarity_threshold=0.8,  # 书名相似度阈值
    content_similarity_threshold=0.85,  # 内容相似度阈值(低于此值需人工审核)
    size_ratio_threshold=0.6  # 大小比例阈值(高于此值才检查内容)
), '', name)
    
    # 特殊处理: 移除"更XX"但保留后面的书名内容
    name = re.sub(r'^更\d+[\s]*', '', name)
    
    # 提取书名核心（去除书名号）
    title_match = re.search(r'[《\[]?([^《》\[\]【】]{2,})[\]》]?', name)
    if title_match:
        title = title_match.group(1)
    else:
        title = name
    
    # 清理书名末尾
    title = re.sub(r'[\s\-_\d]+$', '', title)
    title = title.strip()
    
    # 如果书名太短或为空,使用原始文件名
    if not title or len(title) < 2:
        title = original_name
    
    # 移除书名中的"更XX"
    title = re.sub(r'更\d+', '', title).strip()
    
    return title.lower().strip(), author.lower().strip()


def calculate_title_similarity(title1, title2):
    """计算两个标题的相似度"""
    return SequenceMatcher(None, title1, title2).ratio()


def read_file_sample(filepath, sample_size=2000):
    """读取文件开头部分用于内容比对"""
    try:
        with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read(sample_size)
    except:
        try:
            with open(filepath, 'r', encoding='gbk', errors='ignore') as f:
                return f.read(sample_size)
        except:
            return ""


def calculate_content_similarity(file1, file2):
    """计算两个文件内容的相似度"""
    content1 = read_file_sample(file1)
    content2 = read_file_sample(file2)
    
    if not content1 or not content2:
        return 0.0
    
    return SequenceMatcher(None, content1, content2).ratio()


def find_and_delete_duplicates(folder, 
                               auto_delete=True,
                               similarity_threshold=0.8,
                               content_similarity_threshold=0.85,
                               size_ratio_threshold=0.6):
    """
    查找并删除重复文件
    
    参数:
        folder: 目标文件夹
        auto_delete: True=自动删除高置信度重复, False=全部预览
        similarity_threshold: 书名相似度阈值(0-1)
        content_similarity_threshold: 内容相似度阈值,低于此值需人工审核
        size_ratio_threshold: 大小比例阈值,高于此值需内容检查
    """
    files = [f for f in os.listdir(folder) if f.lower().endswith('.txt')]
    
    if not files:
        print("未找到txt文件")
        return
    
    print(f"{'='*60}")
    print(f"开始处理: {folder}")
    print(f"共找到 {len(files)} 个txt文件")
    print(f"模式: {'自动删除高置信度重复' if auto_delete else '预览模式'}")
    print(f"{'='*60}\n")
    
    # 分组逻辑
    groups = {}
    
    for filename in files:
        title, author = extract_title_author_strict(filename)
        path = os.path.join(folder, filename)
        size = os.path.getsize(path)
        mtime = os.path.getmtime(path)
        
        # 创建分组键 - 必须标题和作者都匹配
        if author != 'unknown':
            key = f"{title}|{author}"
        else:
            # 作者未知的文件,使用完整书名作为键,避免误归组
            # 加入文件名hash确保不同文件不会被归到一起
            key = f"{title}|_noauthor_{hash(filename) % 10000}"
        
        if key not in groups:
            groups[key] = []
        
        groups[key].append({
            'filename': filename,
            'title': title,
            'author': author,
            'path': path,
            'size': size,
            'mtime': mtime
        })
    
    # 统计
    auto_deleted = 0
    manual_review_groups = []
    skipped_groups = []
    
    # 处理每个分组
    for key, file_list in groups.items():
        if len(file_list) == 1:
            continue  # 单文件无需处理
        
        title, author = key.rsplit('|', 1)
        
        # 检查是否为作者未知的误归组
        if author.startswith('_noauthor'):
            skipped_groups.append((title, file_list))
            continue
        
        # 检查书名相似度(针对同作者的多个文件)
        if author != 'unknown':
            titles = [f['title'] for f in file_list]
            if len(titles) > 1:
                # 计算所有配对的平均相似度
                similarities = []
                for i in range(len(titles)):
                    for j in range(i+1, len(titles)):
                        sim = calculate_title_similarity(titles[i], titles[j])
                        similarities.append(sim)
                
                avg_similarity = sum(similarities) / len(similarities) if similarities else 1.0
                
                if avg_similarity < similarity_threshold:
                    # 书名差异过大,可能是不同的书
                    manual_review_groups.append({
                        'reason': f'书名相似度过低({avg_similarity:.2f})',
                        'title': title,
                        'author': author,
                        'files': file_list
                    })
                    continue
        
        # 按大小和修改时间排序
        file_list.sort(key=lambda x: (-x['size'], -x['mtime']))
        main_file = file_list[0]
        
        # 检查其他文件
        has_suspicious = False
        files_to_delete = []
        
        for dup_file in file_list[1:]:
            size_ratio = dup_file['size'] / max(main_file['size'], 1)
            
            # 检查内容相似度
            should_check_content = size_ratio > size_ratio_threshold
            content_sim = -1
            
            if should_check_content:
                content_sim = calculate_content_similarity(main_file['path'], dup_file['path'])
                
                # 内容相似度过低,需要人工审核
                if content_sim < content_similarity_threshold:
                    has_suspicious = True
                    manual_review_groups.append({
                        'reason': f'内容差异较大(相似度={content_sim:.2f})',
                        'title': title,
                        'author': author,
                        'files': [main_file, dup_file]
                    })
                    continue
            
            # 可以安全删除
            files_to_delete.append({
                'file': dup_file,
                'size_ratio': size_ratio,
                'content_sim': content_sim
            })
        
        # 执行删除或预览
        if files_to_delete:
            if auto_delete and not has_suspicious:
                # 自动删除
                print(f"✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    try:
                        os.remove(dup_file['path'])
                        auto_deleted += 1
                        print(f"  ✗ 已删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    except Exception as e:
                        print(f"  ✗ 删除失败: {dup_file['filename']} - {e}")
            else:
                # 预览模式或有可疑情况
                print(f"\n{'='*60}")
                print(f"分组: 书名='{title}' | 作者='{author}'")
                print(f"  ✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    print(f"  ✗ 将删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    if item['content_sim'] >= 0:
                        print(f"     内容相似度: {item['content_sim']:.2f}")
    
    # 输出需要人工审核的分组
    if manual_review_groups:
        print(f"\n{'='*60}")
        print(f"⚠️  需要人工审核的分组 ({len(manual_review_groups)}个):")
        print(f"{'='*60}")
        
        for idx, group in enumerate(manual_review_groups, 1):
            print(f"\n[{idx}] {group['reason']}")
            print(f"    书名: {group['title']}")
            print(f"    作者: {group['author']}")
            for f in group['files']:
                print(f"      - {f['filename']} ({f['size']} bytes)")
    
    # 输出跳过的分组(作者未知且可能误归组)
    if skipped_groups:
        print(f"\n{'='*60}")
        print(f"ℹ️  跳过的分组 ({len(skipped_groups)}个) - 作者未知,已自动分离:")
        print(f"{'='*60}")
        
        for title, file_list in skipped_groups[:5]:  # 只显示前5个
            print(f"\n  书名: {title}")
            for f in file_list:
                print(f"    - {f['filename']}")
        
        if len(skipped_groups) > 5:
            print(f"\n  ... 还有 {len(skipped_groups)-5} 个分组被跳过")
    
    # 总结
    print(f"\n{'='*60}")
    print(f"处理完成!")
    print(f"{'实际' if auto_delete else '预计'}删除文件数: {auto_deleted}")
    print(f"需要人工审核的分组数: {len(manual_review_groups)}")
    print(f"跳过的分组数: {len(skipped_groups)}")
    print(f"{'='*60}\n")


# ========== 执行 ==========

# 直接执行删除,只预览可疑情况
find_and_delete_duplicates(
    folder=r"D:\脚本测试\txt",
    auto_delete=True,  # 自动删除高置信度重复
    similarity_threshold=0.8,  # 书名相似度阈值
    content_similarity_threshold=0.85,  # 内容相似度阈值(低于此值需人工审核)
    size_ratio_threshold=0.6  # 大小比例阈值(高于此值才检查内容)
), '', title)  # 清理末尾
    title = title.strip()
    
    # 如果书名太短或为空,使用原始文件名的前30个字符作为唯一标识
    if not title or len(title) < 2:
        title = original_name[:30]
    
    return title.lower().strip(), author.lower().strip()


def calculate_title_similarity(title1, title2):
    """计算两个标题的相似度"""
    return SequenceMatcher(None, title1, title2).ratio()


def read_file_sample(filepath, sample_size=2000):
    """读取文件开头部分用于内容比对"""
    try:
        with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read(sample_size)
    except:
        try:
            with open(filepath, 'r', encoding='gbk', errors='ignore') as f:
                return f.read(sample_size)
        except:
            return ""


def calculate_content_similarity(file1, file2):
    """计算两个文件内容的相似度"""
    content1 = read_file_sample(file1)
    content2 = read_file_sample(file2)
    
    if not content1 or not content2:
        return 0.0
    
    return SequenceMatcher(None, content1, content2).ratio()


def find_and_delete_duplicates(folder, 
                               auto_delete=True,
                               similarity_threshold=0.8,
                               content_similarity_threshold=0.85,
                               size_ratio_threshold=0.6):
    """
    查找并删除重复文件
    
    参数:
        folder: 目标文件夹
        auto_delete: True=自动删除高置信度重复, False=全部预览
        similarity_threshold: 书名相似度阈值(0-1)
        content_similarity_threshold: 内容相似度阈值,低于此值需人工审核
        size_ratio_threshold: 大小比例阈值,高于此值需内容检查
    """
    files = [f for f in os.listdir(folder) if f.lower().endswith('.txt')]
    
    if not files:
        print("未找到txt文件")
        return
    
    print(f"{'='*60}")
    print(f"开始处理: {folder}")
    print(f"共找到 {len(files)} 个txt文件")
    print(f"模式: {'自动删除高置信度重复' if auto_delete else '预览模式'}")
    print(f"{'='*60}\n")
    
    # 分组逻辑
    groups = {}
    
    for filename in files:
        title, author = extract_title_author_strict(filename)
        path = os.path.join(folder, filename)
        size = os.path.getsize(path)
        mtime = os.path.getmtime(path)
        
        # 创建分组键 - 必须标题和作者都匹配
        if author != 'unknown':
            key = f"{title}|{author}"
        else:
            # 作者未知的文件,使用完整书名作为键,避免误归组
            # 加入文件名hash确保不同文件不会被归到一起
            key = f"{title}|_noauthor_{hash(filename) % 10000}"
        
        if key not in groups:
            groups[key] = []
        
        groups[key].append({
            'filename': filename,
            'title': title,
            'author': author,
            'path': path,
            'size': size,
            'mtime': mtime
        })
    
    # 统计
    auto_deleted = 0
    manual_review_groups = []
    skipped_groups = []
    
    # 处理每个分组
    for key, file_list in groups.items():
        if len(file_list) == 1:
            continue  # 单文件无需处理
        
        title, author = key.rsplit('|', 1)
        
        # 检查是否为作者未知的误归组
        if author.startswith('_noauthor'):
            skipped_groups.append((title, file_list))
            continue
        
        # 检查书名相似度(针对同作者的多个文件)
        if author != 'unknown':
            titles = [f['title'] for f in file_list]
            if len(titles) > 1:
                # 计算所有配对的平均相似度
                similarities = []
                for i in range(len(titles)):
                    for j in range(i+1, len(titles)):
                        sim = calculate_title_similarity(titles[i], titles[j])
                        similarities.append(sim)
                
                avg_similarity = sum(similarities) / len(similarities) if similarities else 1.0
                
                if avg_similarity < similarity_threshold:
                    # 书名差异过大,可能是不同的书
                    manual_review_groups.append({
                        'reason': f'书名相似度过低({avg_similarity:.2f})',
                        'title': title,
                        'author': author,
                        'files': file_list
                    })
                    continue
        
        # 按大小和修改时间排序
        file_list.sort(key=lambda x: (-x['size'], -x['mtime']))
        main_file = file_list[0]
        
        # 检查其他文件
        has_suspicious = False
        files_to_delete = []
        
        for dup_file in file_list[1:]:
            size_ratio = dup_file['size'] / max(main_file['size'], 1)
            
            # 检查内容相似度
            should_check_content = size_ratio > size_ratio_threshold
            content_sim = -1
            
            if should_check_content:
                content_sim = calculate_content_similarity(main_file['path'], dup_file['path'])
                
                # 内容相似度过低,需要人工审核
                if content_sim < content_similarity_threshold:
                    has_suspicious = True
                    manual_review_groups.append({
                        'reason': f'内容差异较大(相似度={content_sim:.2f})',
                        'title': title,
                        'author': author,
                        'files': [main_file, dup_file]
                    })
                    continue
            
            # 可以安全删除
            files_to_delete.append({
                'file': dup_file,
                'size_ratio': size_ratio,
                'content_sim': content_sim
            })
        
        # 执行删除或预览
        if files_to_delete:
            if auto_delete and not has_suspicious:
                # 自动删除
                print(f"✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    try:
                        os.remove(dup_file['path'])
                        auto_deleted += 1
                        print(f"  ✗ 已删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    except Exception as e:
                        print(f"  ✗ 删除失败: {dup_file['filename']} - {e}")
            else:
                # 预览模式或有可疑情况
                print(f"\n{'='*60}")
                print(f"分组: 书名='{title}' | 作者='{author}'")
                print(f"  ✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    print(f"  ✗ 将删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    if item['content_sim'] >= 0:
                        print(f"     内容相似度: {item['content_sim']:.2f}")
    
    # 输出需要人工审核的分组
    if manual_review_groups:
        print(f"\n{'='*60}")
        print(f"⚠️  需要人工审核的分组 ({len(manual_review_groups)}个):")
        print(f"{'='*60}")
        
        for idx, group in enumerate(manual_review_groups, 1):
            print(f"\n[{idx}] {group['reason']}")
            print(f"    书名: {group['title']}")
            print(f"    作者: {group['author']}")
            for f in group['files']:
                print(f"      - {f['filename']} ({f['size']} bytes)")
    
    # 输出跳过的分组(作者未知且可能误归组)
    if skipped_groups:
        print(f"\n{'='*60}")
        print(f"  跳过的分组 ({len(skipped_groups)}个) - 作者未知,已自动分离:")
        print(f"{'='*60}")
        
        for title, file_list in skipped_groups[:5]:  # 只显示前5个
            print(f"\n  书名: {title}")
            for f in file_list:
                print(f"    - {f['filename']}")
        
        if len(skipped_groups) > 5:
            print(f"\n  ... 还有 {len(skipped_groups)-5} 个分组被跳过")
    
    # 总结
    print(f"\n{'='*60}")
    print(f"处理完成!")
    print(f"{'实际' if auto_delete else '预计'}删除文件数: {auto_deleted}")
    print(f"需要人工审核的分组数: {len(manual_review_groups)}")
    print(f"跳过的分组数: {len(skipped_groups)}")
    print(f"{'='*60}\n")


# ========== 执行 ==========

# 直接执行删除,只预览可疑情况
find_and_delete_duplicates(
    folder=r"D:\脚本测试\txt",
    auto_delete=True,  # 自动删除高置信度重复
    similarity_threshold=0.8,  # 书名相似度阈值
    content_similarity_threshold=0.85,  # 内容相似度阈值(低于此值需人工审核)
    size_ratio_threshold=0.6  # 大小比例阈值(高于此值才检查内容)
), '', name)
    name = re.sub(r'[\s]*(?:txt|TXT)[\s]*
    
    # 提取书名核心（去除书名号）
    title_match = re.search(r'[《\[]?([^《》\[\]【】]{2,})[\]》]?', name)
    if title_match:
        title = title_match.group(1)
    else:
        title = name
    
    # 清理书名末尾
    title = re.sub(r'[\s\-_\d]+$', '', title)
    title = title.strip()
    
    # 如果书名太短或为空,使用原始文件名
    if not title or len(title) < 2:
        title = original_name
    
    # 移除书名中的"更XX"
    title = re.sub(r'更\d+', '', title).strip()
    
    return title.lower().strip(), author.lower().strip()


def calculate_title_similarity(title1, title2):
    """计算两个标题的相似度"""
    return SequenceMatcher(None, title1, title2).ratio()


def read_file_sample(filepath, sample_size=2000):
    """读取文件开头部分用于内容比对"""
    try:
        with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read(sample_size)
    except:
        try:
            with open(filepath, 'r', encoding='gbk', errors='ignore') as f:
                return f.read(sample_size)
        except:
            return ""


def calculate_content_similarity(file1, file2):
    """计算两个文件内容的相似度"""
    content1 = read_file_sample(file1)
    content2 = read_file_sample(file2)
    
    if not content1 or not content2:
        return 0.0
    
    return SequenceMatcher(None, content1, content2).ratio()


def find_and_delete_duplicates(folder, 
                               auto_delete=True,
                               similarity_threshold=0.8,
                               content_similarity_threshold=0.85,
                               size_ratio_threshold=0.6):
    """
    查找并删除重复文件
    
    参数:
        folder: 目标文件夹
        auto_delete: True=自动删除高置信度重复, False=全部预览
        similarity_threshold: 书名相似度阈值(0-1)
        content_similarity_threshold: 内容相似度阈值,低于此值需人工审核
        size_ratio_threshold: 大小比例阈值,高于此值需内容检查
    """
    files = [f for f in os.listdir(folder) if f.lower().endswith('.txt')]
    
    if not files:
        print("未找到txt文件")
        return
    
    print(f"{'='*60}")
    print(f"开始处理: {folder}")
    print(f"共找到 {len(files)} 个txt文件")
    print(f"模式: {'自动删除高置信度重复' if auto_delete else '预览模式'}")
    print(f"{'='*60}\n")
    
    # 分组逻辑
    groups = {}
    
    for filename in files:
        title, author = extract_title_author_strict(filename)
        path = os.path.join(folder, filename)
        size = os.path.getsize(path)
        mtime = os.path.getmtime(path)
        
        # 创建分组键 - 必须标题和作者都匹配
        if author != 'unknown':
            key = f"{title}|{author}"
        else:
            # 作者未知的文件,使用完整书名作为键,避免误归组
            # 加入文件名hash确保不同文件不会被归到一起
            key = f"{title}|_noauthor_{hash(filename) % 10000}"
        
        if key not in groups:
            groups[key] = []
        
        groups[key].append({
            'filename': filename,
            'title': title,
            'author': author,
            'path': path,
            'size': size,
            'mtime': mtime
        })
    
    # 统计
    auto_deleted = 0
    manual_review_groups = []
    skipped_groups = []
    
    # 处理每个分组
    for key, file_list in groups.items():
        if len(file_list) == 1:
            continue  # 单文件无需处理
        
        title, author = key.rsplit('|', 1)
        
        # 检查是否为作者未知的误归组
        if author.startswith('_noauthor'):
            skipped_groups.append((title, file_list))
            continue
        
        # 检查书名相似度(针对同作者的多个文件)
        if author != 'unknown':
            titles = [f['title'] for f in file_list]
            if len(titles) > 1:
                # 计算所有配对的平均相似度
                similarities = []
                for i in range(len(titles)):
                    for j in range(i+1, len(titles)):
                        sim = calculate_title_similarity(titles[i], titles[j])
                        similarities.append(sim)
                
                avg_similarity = sum(similarities) / len(similarities) if similarities else 1.0
                
                if avg_similarity < similarity_threshold:
                    # 书名差异过大,可能是不同的书
                    manual_review_groups.append({
                        'reason': f'书名相似度过低({avg_similarity:.2f})',
                        'title': title,
                        'author': author,
                        'files': file_list
                    })
                    continue
        
        # 按大小和修改时间排序
        file_list.sort(key=lambda x: (-x['size'], -x['mtime']))
        main_file = file_list[0]
        
        # 检查其他文件
        has_suspicious = False
        files_to_delete = []
        
        for dup_file in file_list[1:]:
            size_ratio = dup_file['size'] / max(main_file['size'], 1)
            
            # 检查内容相似度
            should_check_content = size_ratio > size_ratio_threshold
            content_sim = -1
            
            if should_check_content:
                content_sim = calculate_content_similarity(main_file['path'], dup_file['path'])
                
                # 内容相似度过低,需要人工审核
                if content_sim < content_similarity_threshold:
                    has_suspicious = True
                    manual_review_groups.append({
                        'reason': f'内容差异较大(相似度={content_sim:.2f})',
                        'title': title,
                        'author': author,
                        'files': [main_file, dup_file]
                    })
                    continue
            
            # 可以安全删除
            files_to_delete.append({
                'file': dup_file,
                'size_ratio': size_ratio,
                'content_sim': content_sim
            })
        
        # 执行删除或预览
        if files_to_delete:
            if auto_delete and not has_suspicious:
                # 自动删除
                print(f"✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    try:
                        os.remove(dup_file['path'])
                        auto_deleted += 1
                        print(f"  ✗ 已删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    except Exception as e:
                        print(f"  ✗ 删除失败: {dup_file['filename']} - {e}")
            else:
                # 预览模式或有可疑情况
                print(f"\n{'='*60}")
                print(f"分组: 书名='{title}' | 作者='{author}'")
                print(f"  ✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    print(f"  ✗ 将删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    if item['content_sim'] >= 0:
                        print(f"     内容相似度: {item['content_sim']:.2f}")
    
    # 输出需要人工审核的分组
    if manual_review_groups:
        print(f"\n{'='*60}")
        print(f"! 需要人工审核的分组 ({len(manual_review_groups)}个):")
        print(f"{'='*60}")
        
        for idx, group in enumerate(manual_review_groups, 1):
            print(f"\n[{idx}] {group['reason']}")
            print(f"    书名: {group['title']}")
            print(f"    作者: {group['author']}")
            for f in group['files']:
                print(f"      - {f['filename']} ({f['size']} bytes)")
    
    # 输出跳过的分组(作者未知且可能误归组)
    if skipped_groups:
        print(f"\n{'='*60}")
        print(f"  跳过的分组 ({len(skipped_groups)}个) - 作者未知,已自动分离:")
        print(f"{'='*60}")
        
        for title, file_list in skipped_groups[:5]:  # 只显示前5个
            print(f"\n  书名: {title}")
            for f in file_list:
                print(f"    - {f['filename']}")
        
        if len(skipped_groups) > 5:
            print(f"\n  ... 还有 {len(skipped_groups)-5} 个分组被跳过")
    
    # 总结
    print(f"\n{'='*60}")
    print(f"处理完成!")
    print(f"{'实际' if auto_delete else '预计'}删除文件数: {auto_deleted}")
    print(f"需要人工审核的分组数: {len(manual_review_groups)}")
    print(f"跳过的分组数: {len(skipped_groups)}")
    print(f"{'='*60}\n")


# ========== 执行 ==========

# 直接执行删除,只预览可疑情况
find_and_delete_duplicates(
    folder=r"D:\脚本测试\txt",
    auto_delete=True,  # 自动删除高置信度重复
    similarity_threshold=0.8,  # 书名相似度阈值
    content_similarity_threshold=0.85,  # 内容相似度阈值(低于此值需人工审核)
    size_ratio_threshold=0.6  # 大小比例阈值(高于此值才检查内容)
), '', name)
    
    # 特殊处理: 移除"更XX"但保留后面的书名内容
    name = re.sub(r'^更\d+[\s]*', '', name)
    
    # 提取书名核心（去除书名号）
    title_match = re.search(r'[《\[]?([^《》\[\]【】]{2,})[\]》]?', name)
    if title_match:
        title = title_match.group(1)
    else:
        title = name
    
    # 清理书名末尾
    title = re.sub(r'[\s\-_\d]+$', '', title)
    title = title.strip()
    
    # 如果书名太短或为空,使用原始文件名
    if not title or len(title) < 2:
        title = original_name
    
    # 移除书名中的"更XX"
    title = re.sub(r'更\d+', '', title).strip()
    
    return title.lower().strip(), author.lower().strip()


def calculate_title_similarity(title1, title2):
    """计算两个标题的相似度"""
    return SequenceMatcher(None, title1, title2).ratio()


def read_file_sample(filepath, sample_size=2000):
    """读取文件开头部分用于内容比对"""
    try:
        with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read(sample_size)
    except:
        try:
            with open(filepath, 'r', encoding='gbk', errors='ignore') as f:
                return f.read(sample_size)
        except:
            return ""


def calculate_content_similarity(file1, file2):
    """计算两个文件内容的相似度"""
    content1 = read_file_sample(file1)
    content2 = read_file_sample(file2)
    
    if not content1 or not content2:
        return 0.0
    
    return SequenceMatcher(None, content1, content2).ratio()


def find_and_delete_duplicates(folder, 
                               auto_delete=True,
                               similarity_threshold=0.8,
                               content_similarity_threshold=0.85,
                               size_ratio_threshold=0.6):
    """
    查找并删除重复文件
    
    参数:
        folder: 目标文件夹
        auto_delete: True=自动删除高置信度重复, False=全部预览
        similarity_threshold: 书名相似度阈值(0-1)
        content_similarity_threshold: 内容相似度阈值,低于此值需人工审核
        size_ratio_threshold: 大小比例阈值,高于此值需内容检查
    """
    files = [f for f in os.listdir(folder) if f.lower().endswith('.txt')]
    
    if not files:
        print("未找到txt文件")
        return
    
    print(f"{'='*60}")
    print(f"开始处理: {folder}")
    print(f"共找到 {len(files)} 个txt文件")
    print(f"模式: {'自动删除高置信度重复' if auto_delete else '预览模式'}")
    print(f"{'='*60}\n")
    
    # 分组逻辑
    groups = {}
    
    for filename in files:
        title, author = extract_title_author_strict(filename)
        path = os.path.join(folder, filename)
        size = os.path.getsize(path)
        mtime = os.path.getmtime(path)
        
        # 创建分组键 - 必须标题和作者都匹配
        if author != 'unknown':
            key = f"{title}|{author}"
        else:
            # 作者未知的文件,使用完整书名作为键,避免误归组
            # 加入文件名hash确保不同文件不会被归到一起
            key = f"{title}|_noauthor_{hash(filename) % 10000}"
        
        if key not in groups:
            groups[key] = []
        
        groups[key].append({
            'filename': filename,
            'title': title,
            'author': author,
            'path': path,
            'size': size,
            'mtime': mtime
        })
    
    # 统计
    auto_deleted = 0
    manual_review_groups = []
    skipped_groups = []
    
    # 处理每个分组
    for key, file_list in groups.items():
        if len(file_list) == 1:
            continue  # 单文件无需处理
        
        title, author = key.rsplit('|', 1)
        
        # 检查是否为作者未知的误归组
        if author.startswith('_noauthor'):
            skipped_groups.append((title, file_list))
            continue
        
        # 检查书名相似度(针对同作者的多个文件)
        if author != 'unknown':
            titles = [f['title'] for f in file_list]
            if len(titles) > 1:
                # 计算所有配对的平均相似度
                similarities = []
                for i in range(len(titles)):
                    for j in range(i+1, len(titles)):
                        sim = calculate_title_similarity(titles[i], titles[j])
                        similarities.append(sim)
                
                avg_similarity = sum(similarities) / len(similarities) if similarities else 1.0
                
                if avg_similarity < similarity_threshold:
                    # 书名差异过大,可能是不同的书
                    manual_review_groups.append({
                        'reason': f'书名相似度过低({avg_similarity:.2f})',
                        'title': title,
                        'author': author,
                        'files': file_list
                    })
                    continue
        
        # 按大小和修改时间排序
        file_list.sort(key=lambda x: (-x['size'], -x['mtime']))
        main_file = file_list[0]
        
        # 检查其他文件
        has_suspicious = False
        files_to_delete = []
        
        for dup_file in file_list[1:]:
            size_ratio = dup_file['size'] / max(main_file['size'], 1)
            
            # 检查内容相似度
            should_check_content = size_ratio > size_ratio_threshold
            content_sim = -1
            
            if should_check_content:
                content_sim = calculate_content_similarity(main_file['path'], dup_file['path'])
                
                # 内容相似度过低,需要人工审核
                if content_sim < content_similarity_threshold:
                    has_suspicious = True
                    manual_review_groups.append({
                        'reason': f'内容差异较大(相似度={content_sim:.2f})',
                        'title': title,
                        'author': author,
                        'files': [main_file, dup_file]
                    })
                    continue
            
            # 可以安全删除
            files_to_delete.append({
                'file': dup_file,
                'size_ratio': size_ratio,
                'content_sim': content_sim
            })
        
        # 执行删除或预览
        if files_to_delete:
            if auto_delete and not has_suspicious:
                # 自动删除
                print(f"✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    try:
                        os.remove(dup_file['path'])
                        auto_deleted += 1
                        print(f"  ✗ 已删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    except Exception as e:
                        print(f"  ✗ 删除失败: {dup_file['filename']} - {e}")
            else:
                # 预览模式或有可疑情况
                print(f"\n{'='*60}")
                print(f"分组: 书名='{title}' | 作者='{author}'")
                print(f"  ✓ 保留: {main_file['filename']} ({main_file['size']} bytes)")
                for item in files_to_delete:
                    dup_file = item['file']
                    print(f"  ✗ 将删除: {dup_file['filename']} ({dup_file['size']} bytes)")
                    if item['content_sim'] >= 0:
                        print(f"     内容相似度: {item['content_sim']:.2f}")
    
    # 输出需要人工审核的分组
    if manual_review_groups:
        print(f"\n{'='*60}")
        print(f"⚠️  需要人工审核的分组 ({len(manual_review_groups)}个):")
        print(f"{'='*60}")
        
        for idx, group in enumerate(manual_review_groups, 1):
            print(f"\n[{idx}] {group['reason']}")
            print(f"    书名: {group['title']}")
            print(f"    作者: {group['author']}")
            for f in group['files']:
                print(f"      - {f['filename']} ({f['size']} bytes)")
    
    # 输出跳过的分组(作者未知且可能误归组)
    if skipped_groups:
        print(f"\n{'='*60}")
        print(f"ℹ️  跳过的分组 ({len(skipped_groups)}个) - 作者未知,已自动分离:")
        print(f"{'='*60}")
        
        for title, file_list in skipped_groups[:5]:  # 只显示前5个
            print(f"\n  书名: {title}")
            for f in file_list:
                print(f"    - {f['filename']}")
        
        if len(skipped_groups) > 5:
            print(f"\n  ... 还有 {len(skipped_groups)-5} 个分组被跳过")
    
    # 总结
    print(f"\n{'='*60}")
    print(f"处理完成!")
    print(f"{'实际' if auto_delete else '预计'}删除文件数: {auto_deleted}")
    print(f"需要人工审核的分组数: {len(manual_review_groups)}")
    print(f"跳过的分组数: {len(skipped_groups)}")
    print(f"{'='*60}\n")


# ========== 执行 ==========

# 直接执行删除,只预览可疑情况
find_and_delete_duplicates(
    folder=r"D:\脚本测试\txt",
    auto_delete=True,  # 自动删除高置信度重复
    similarity_threshold=0.8,  # 书名相似度阈值
    content_similarity_threshold=0.85,  # 内容相似度阈值(低于此值需人工审核)
    size_ratio_threshold=0.6  # 大小比例阈值(高于此值才检查内容)
)
