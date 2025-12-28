<!-- 
  改进的 Markdown 渲染组件（使用 marked.js）
  解决图片闪烁和标签不完整的问题
  核心策略：
  1. 增量渲染：只处理新增内容，避免重新渲染已存在的内容
  2. 按块级元素分割：不按图片分割，而是按完整的 HTML 块分割
  3. 稳定的 key：每个块使用唯一稳定的 ID，Vue 不会重新渲染已存在的块
-->
<template>
    <div class="markdown" ref="markdownContainer">
        <!-- 使用稳定的 key，避免重新渲染已存在的块 -->
        <div 
            v-for="block in renderBlocks" 
            :key="block.id" 
            class="markdown-block"
            v-html="block.html"
        ></div>
    </div>
</template>

<script>
import { marked } from 'marked';
import DOMPurify from 'dompurify';

export default {
    name: 'MarkdownContent',
    props: {
        content: { 
            type: String, 
            default: '' 
        },
        isCompleted: { 
            type: Boolean, 
            default: false 
        }
    },
    data() {
        return {
            // 存储已渲染的块，每个块都有稳定的 ID
            renderBlocks: [],
            // 当前已处理的内容长度，用于增量更新
            processedLength: 0,
            // 块计数器，用于生成唯一 ID
            blockCounter: 0,
            // 累积的 markdown 源码（用于延迟渲染）
            accumulatedMarkdown: ''
        };
    },
    watch: {
        content: {
            immediate: true,
            handler(newContent, oldContent) {
                this.handleContentUpdate(newContent);
            }
        },
        isCompleted: {
            handler(isCompleted) {
                // 当内容完成时，渲染剩余的累积内容
                if (isCompleted && this.accumulatedMarkdown) {
                    if (this.accumulatedMarkdown.trim()) {
                        this.appendNewContent(this.accumulatedMarkdown);
                    }
                    this.accumulatedMarkdown = '';
                }
            }
        }
    },
    methods: {
        /**
         * 处理内容更新，使用智能增量渲染策略
         * 累积 markdown 源码，在块级边界处进行渲染，避免碎片化
         */
        handleContentUpdate(newContent) {
            if (!newContent) {
                this.renderBlocks = [];
                this.processedLength = 0;
                this.blockCounter = 0;
                this.accumulatedMarkdown = '';
                return;
            }

            // 如果内容长度减少，说明是重置，重新渲染
            if (newContent.length < this.processedLength) {
                this.renderBlocks = [];
                this.processedLength = 0;
                this.blockCounter = 0;
                this.accumulatedMarkdown = '';
            }

            // 获取新增的内容部分
            const newPart = newContent.substring(this.processedLength);
            
            if (newPart) {
                // 累积新增内容
                this.accumulatedMarkdown += newPart;
                
                // 检测块级边界，在边界处进行渲染
                this.processAccumulatedContent();
            }

            this.processedLength = newContent.length;
        },

        /**
         * 处理累积的 markdown 内容，在块级边界处进行渲染
         */
        processAccumulatedContent() {
            if (!this.accumulatedMarkdown) return;

            // 检测块级边界（段落结束、列表项结束、标题结束等）
            const blockBoundaries = this.detectBlockBoundaries(this.accumulatedMarkdown);
            
            if (blockBoundaries.length > 0) {
                // 有完整的块，渲染这些块
                let lastIndex = 0;
                blockBoundaries.forEach((boundaryIndex) => {
                    const blockMarkdown = this.accumulatedMarkdown.substring(lastIndex, boundaryIndex + 1);
                    if (blockMarkdown.trim()) {
                        this.appendNewContent(blockMarkdown);
                    }
                    lastIndex = boundaryIndex + 1;
                });
                
                // 保留未完成的块
                this.accumulatedMarkdown = this.accumulatedMarkdown.substring(lastIndex);
            }
            
            // 如果内容很长但没有块级边界，强制渲染一部分（避免内存问题）
            if (this.accumulatedMarkdown.length > 1000) {
                // 尝试在最近的换行处分割
                const lastNewline = this.accumulatedMarkdown.lastIndexOf('\n');
                if (lastNewline > 100) {
                    const blockMarkdown = this.accumulatedMarkdown.substring(0, lastNewline + 1);
                    this.appendNewContent(blockMarkdown);
                    this.accumulatedMarkdown = this.accumulatedMarkdown.substring(lastNewline + 1);
                }
            }
        },

        /**
         * 检测 markdown 中的块级边界
         * 返回所有完整块的结束位置索引数组
         */
        detectBlockBoundaries(markdown) {
            const boundaries = [];
            const lines = markdown.split('\n');
            const state = {
                currentIndex: 0,
                inCodeBlock: false,
                codeBlockFence: '',
                inList: false,
                listIndent: -1,
                lastNonEmptyLineIndex: -1
            };
            
            for (let i = 0; i < lines.length; i++) {
                const line = lines[i];
                const lineStartIndex = state.currentIndex;
                const lineEndIndex = state.currentIndex + line.length;
                const isEmptyLine = line.trim() === '';
                
                // 处理代码块
                if (this.processCodeBlock(line, i, lineEndIndex, state, boundaries, lineStartIndex)) {
                    state.currentIndex = lineEndIndex + 1;
                    continue;
                }
                
                if (state.inCodeBlock) {
                    state.currentIndex = lineEndIndex + 1;
                    continue;
                }
                
                // 处理标题
                if (this.processHeading(line, i, lineEndIndex, state, boundaries, lineStartIndex)) {
                    state.currentIndex = lineEndIndex + 1;
                    continue;
                }
                
                // 处理列表项
                if (this.processListItem(line, i, isEmptyLine, lineEndIndex, state, boundaries, lineStartIndex)) {
                    state.currentIndex = lineEndIndex + 1;
                    continue;
                }
                
                // 处理空行（块级边界）
                this.processEmptyLine(line, i, isEmptyLine, lines, lineStartIndex, state, boundaries);
                
                state.currentIndex = lineEndIndex + 1;
            }
            
            return boundaries;
        },

        /**
         * 处理代码块检测
         */
        processCodeBlock(line, lineIndex, lineEndIndex, state, boundaries, lineStartIndex) {
            const codeBlockMatch = line.match(/^(\s*)(```|~~~)/);
            if (!codeBlockMatch) return false;
            
            if (!state.inCodeBlock) {
                // 代码块开始
                if (state.lastNonEmptyLineIndex >= 0 && state.lastNonEmptyLineIndex < lineIndex - 1) {
                    boundaries.push(state.currentIndex - 1);
                }
                state.inCodeBlock = true;
                state.codeBlockFence = codeBlockMatch[2];
            } else if (codeBlockMatch[2] === state.codeBlockFence) {
                // 代码块结束
                state.inCodeBlock = false;
                state.codeBlockFence = '';
                boundaries.push(lineEndIndex);
                state.lastNonEmptyLineIndex = lineIndex;
                return true;
            }
            return false;
        },

        /**
         * 处理标题检测
         */
        processHeading(line, lineIndex, lineEndIndex, state, boundaries, lineStartIndex) {
            if (!/^#{1,6}\s+/.test(line) || state.inList) return false;
            
            if (state.lastNonEmptyLineIndex >= 0 && state.lastNonEmptyLineIndex < lineIndex - 1) {
                boundaries.push(lineStartIndex - 1);
            }
            boundaries.push(lineEndIndex);
            state.lastNonEmptyLineIndex = lineIndex;
            return true;
        },

        /**
         * 处理列表项检测
         */
        processListItem(line, lineIndex, isEmptyLine, lineEndIndex, state, boundaries, lineStartIndex) {
            if (isEmptyLine) return false;
            
            const listMatch = line.match(/^(\s*)([-*+]|\d+\.)\s+/);
            if (!listMatch) return false;
            
            const indent = listMatch[1].length;
            if (!state.inList || indent < state.listIndent) {
                // 新列表开始，或列表层级变化（降级）
                if (state.inList && state.lastNonEmptyLineIndex >= 0) {
                    boundaries.push(lineStartIndex - 1);
                }
                state.inList = true;
                state.listIndent = indent;
            }
            state.lastNonEmptyLineIndex = lineIndex;
            return true;
        },

        /**
         * 处理空行（块级边界）
         */
        processEmptyLine(line, lineIndex, isEmptyLine, lines, lineStartIndex, state, boundaries) {
            if (!isEmptyLine) {
                // 非空行
                if (!state.inList) {
                    state.lastNonEmptyLineIndex = lineIndex;
                }
                return;
            }
            
            // 空行处理
            if (state.lastNonEmptyLineIndex >= 0 && state.lastNonEmptyLineIndex < lineIndex - 1) {
                boundaries.push(lineStartIndex - 1);
            }
            
            if (state.inList) {
                // 列表在空行后结束（除非下一行继续列表）
                if (lineIndex < lines.length - 1) {
                    const nextLine = lines[lineIndex + 1];
                    const nextListMatch = nextLine.match(/^(\s*)([-*+]|\d+\.)\s+/);
                    const nextIndent = nextListMatch ? nextListMatch[1].length : -1;
                    if (!nextListMatch || nextIndent < state.listIndent) {
                        state.inList = false;
                        state.listIndent = -1;
                    }
                } else {
                    state.inList = false;
                    state.listIndent = -1;
                }
            }
        },

        /**
         * 追加新内容，按块级元素分割以保持 HTML 标签完整性
         * 智能合并未完成的段落，避免句子被分割
         */
        appendNewContent(newPart) {
            try {
                // 使用 marked 渲染新内容
                const renderer = new marked.Renderer();
                const html = marked(newPart, { 
                    renderer, 
                    headerIds: false, 
                    mangle: false, 
                    breaks: true, 
                    gfm: true, 
                    tables: true, 
                    sanitize: false 
                });

                // 清理 HTML
                const cleanHtml = DOMPurify.sanitize(html, {
                    ALLOWED_TAGS: [
                        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
                        'p', 'br', 'strong', 'em', 'u', 'del',
                        'ul', 'ol', 'li', 'blockquote', 'pre', 'code',
                        'table', 'thead', 'tbody', 'tr', 'th', 'td',
                        'a', 'img', 'hr', 'input', 'label', 'div', 'span'
                    ],
                    ALLOWED_ATTR: [
                        'href', 'title', 'alt', 'src', 'class', 'id',
                        'type', 'style', 'checked', 'disabled', 'target',
                        'rel', 'data-placeholder-id', 'loading'
                    ],
                    ALLOWED_DATA_ATTR: false,
                    ALLOWED_UNKNOWN_PROTOCOLS: false
                });

                // 添加表格样式
                const styledHtml = cleanHtml.replace(/<table>/g, '<table class="markdown-table">');

                // 按块级元素分割，确保每个块都是完整的 HTML 结构
                const blocks = this.splitIntoCompleteBlocks(styledHtml);
                
                // 智能合并未完成的块（段落、列表、表格等）
                blocks.forEach((blockHtml) => {
                    if (blockHtml.trim()) {
                        const lastBlock = this.renderBlocks[this.renderBlocks.length - 1];
                        
                        // 尝试合并到最后一个块
                        // 重要：如果最后一个块包含图片，不要合并，避免图片闪烁
                        if (lastBlock && this.shouldMergeBlocks(lastBlock.html, blockHtml)) {
                            // 检查最后一个块是否包含图片
                            if (this.containsImage(lastBlock.html)) {
                                // 如果包含图片，创建新块而不是合并，避免图片闪烁
                                const blockId = `md-block-${this.blockCounter++}`;
                                this.renderBlocks.push({
                                    id: blockId,
                                    html: blockHtml
                                });
                            } else {
                                // 不包含图片，可以安全合并
                                const mergedBlock = this.mergeBlocks(lastBlock, blockHtml);
                                if (mergedBlock) {
                                    // 更新最后一个块
                                    Object.assign(lastBlock, mergedBlock);
                                }
                            }
                        } else {
                            // 否则，创建新块
                            const blockId = `md-block-${this.blockCounter++}`;
                            this.renderBlocks.push({
                                id: blockId,
                                html: blockHtml
                            });
                        }
                    }
                });
            } catch (error) {
                console.error('Markdown rendering error:', error);
                // 出错时直接添加原始内容
                const blockId = `md-block-${this.blockCounter++}`;
                this.renderBlocks.push({
                    id: blockId,
                    html: DOMPurify.sanitize(newPart)
                });
            }
        },

        /**
         * 判断是否应该合并两个块
         * 合并规则：
         * 1. 段落：如果最后一个块是未完成的段落，且当前块也是段落，则合并
         * 2. 列表：如果最后一个块是未完成的列表，且当前块是列表项、列表或段落（需要转换为列表项），则合并
         * 3. 表格：如果最后一个块是未完成的表格，且当前块是表格行或表格，则合并
         * 4. 引用块：如果最后一个块是未完成的引用块，且当前块也是引用块内容，则合并
         */
        shouldMergeBlocks(lastBlockHtml, currentBlockHtml) {
            if (!lastBlockHtml || !currentBlockHtml) return false;

            // 1. 段落合并
            if (this.isParagraphBlock(lastBlockHtml) && this.isParagraphBlock(currentBlockHtml)) {
                return !this.isParagraphComplete(lastBlockHtml);
            }

            // 2. 列表合并
            if (this.isIncompleteList(lastBlockHtml)) {
                // 如果最后一个块是未完成的列表，且当前块是列表项、列表或段落（可能是列表项被渲染成了段落），则合并
                return this.isListItemBlock(currentBlockHtml) || 
                       this.isListBlock(currentBlockHtml) || 
                       this.isParagraphBlock(currentBlockHtml);
            }

            // 3. 表格合并
            if (this.isIncompleteTable(lastBlockHtml)) {
                // 如果最后一个块是未完成的表格，且当前块是表格行或表格，则合并
                return this.isTableRowBlock(currentBlockHtml) || this.isTableBlock(currentBlockHtml);
            }

            // 4. 引用块合并
            if (this.isIncompleteBlockquote(lastBlockHtml)) {
                return this.isBlockquoteContent(currentBlockHtml);
            }

            return false;
        },

        /**
         * 合并两个块
         */
        mergeBlocks(lastBlock, currentBlockHtml) {
            const lastHtml = lastBlock.html;
            const currentHtml = currentBlockHtml;
            const updatedBlock = {...lastBlock};

            // 1. 段落合并
            if (this.isParagraphBlock(lastHtml) && this.isParagraphBlock(currentHtml)) {
                const lastContent = lastHtml.replace(/<\/p>\s*$/, '');
                const currentContent = currentHtml.replace(/^<p[^>]*>/, '').replace(/<\/p>\s*$/, '');
                updatedBlock.html = lastContent + currentContent + '</p>';
                return updatedBlock;
            }

            // 2. 列表合并
            if (this.isIncompleteList(lastHtml)) {
                // 确定列表类型（ul 或 ol）
                const listTypeMatch = lastHtml.match(/<([uo]l)(?:\s[^>]*)?>/i);
                const listType = listTypeMatch ? listTypeMatch[1].toLowerCase() : 'ul';
                const listTag = listType === 'ul' ? 'ul' : 'ol';

                if (this.isListItemBlock(currentHtml)) {
                    // 当前块是列表项，直接追加到列表
                    const lastContent = lastHtml.replace(/<\/[uo]l>/gi, '');
                    updatedBlock.html = lastContent + currentHtml + `</${listTag}>`;
                } else if (this.isListBlock(currentHtml)) {
                    // 当前块是列表，合并列表项
                    const listItems = currentHtml.match(/<li[^>]*>[\s\S]*?<\/li>/gi) || [];
                    const lastContent = lastHtml.replace(/<\/[uo]l>/gi, '');
                    updatedBlock.html = lastContent + listItems.join('') + `</${listTag}>`;
                } else if (this.isParagraphBlock(currentHtml)) {
                    // 当前块是段落，但应该作为列表项（marked 可能将列表项渲染成了段落）
                    // 提取段落内容，转换为列表项
                    const paragraphContent = currentHtml.replace(/^<p[^>]*>/, '').replace(/<\/p>\s*$/, '');
                    const lastContent = lastHtml.replace(/<\/[uo]l>/gi, '');
                    updatedBlock.html = lastContent + `<li>${paragraphContent}</li></${listTag}>`;
                }
                return updatedBlock;
            }

            // 3. 表格合并
            if (this.isIncompleteTable(lastHtml)) {
                if (this.isTableRowBlock(currentHtml)) {
                    // 当前块是表格行，追加到表格
                    const lastContent = lastHtml.replace(/<\/table>\s*$/i, '');
                    updatedBlock.html = lastContent + currentHtml + '</table>';
                } else if (this.isTableBlock(currentHtml)) {
                    // 当前块是表格，合并行
                    const tableRows = currentHtml.match(/<tr[^>]*>[\s\S]*?<\/tr>/gi) || [];
                    const lastContent = lastHtml.replace(/<\/table>\s*$/i, '');
                    updatedBlock.html = lastContent + tableRows.join('') + '</table>';
                }
                return updatedBlock;
            }

            // 4. 引用块合并
            if (this.isIncompleteBlockquote(lastHtml)) {
                const lastContent = lastHtml.replace(/<\/blockquote>\s*$/i, '');
                let currentContent = currentHtml;
                // 移除当前块的 blockquote 标签（如果有）
                currentContent = currentContent.replace(/^<blockquote[^>]*>/i, '').replace(/<\/blockquote>\s*$/i, '');
                updatedBlock.html = lastContent + currentContent + '</blockquote>';
                return updatedBlock;
            }
        },

        /**
         * 判断一个 HTML 块是否是段落（<p> 标签）
         */
        isParagraphBlock(html) {
            if (!html) return false;
            const trimmed = html.trim();
            return /^<p(?:\s[^>]*)?>/.test(trimmed) && /<\/p>\s*$/.test(trimmed);
        },

        /**
         * 判断段落是否已完成（以句号、问号、感叹号等结束）
         */
        isParagraphComplete(html) {
            if (!html) return false;
            const textContent = html.replace(/<[^>]+>/g, '').trim();
            return /[。！？.!?]\s*$/.test(textContent) || /<img/.test(html);
        },

        /**
         * 判断是否是未完成的列表（有 <ul> 或 <ol> 但没有对应的 </ul> 或 </ol>）
         */
        isIncompleteList(html) {
            if (!html) return false;
            const trimmed = html.trim();
            const hasListStart = /<[uo]l(?:\s[^>]*)?>/i.test(trimmed);
            const hasListEnd = /<\/[uo]l>/i.test(trimmed);
            return hasListStart && !hasListEnd;
        },

        /**
         * 判断是否是列表项块（<li> 标签）
         */
        isListItemBlock(html) {
            if (!html) return false;
            const trimmed = html.trim();
            return /^<li(?:\s[^>]*)?>/.test(trimmed) && /<\/li>\s*$/.test(trimmed);
        },

        /**
         * 判断是否是列表块（<ul> 或 <ol> 标签）
         */
        isListBlock(html) {
            if (!html) return false;
            const trimmed = html.trim();
            return /^<[uo]l(?:\s[^>]*)?>/i.test(trimmed) && /<\/[uo]l>\s*$/i.test(trimmed);
        },

        /**
         * 判断是否是未完成的表格（有 <table> 但没有 </table>）
         */
        isIncompleteTable(html) {
            if (!html) return false;
            const trimmed = html.trim();
            const hasTableStart = /<table(?:\s[^>]*)?>/i.test(trimmed);
            const hasTableEnd = /<\/table>/i.test(trimmed);
            return hasTableStart && !hasTableEnd;
        },

        /**
         * 判断是否是表格行块（<tr> 标签）
         */
        isTableRowBlock(html) {
            if (!html) return false;
            const trimmed = html.trim();
            return /^<tr(?:\s[^>]*)?>/.test(trimmed) && /<\/tr>\s*$/.test(trimmed);
        },

        /**
         * 判断是否是表格块（<table> 标签）
         */
        isTableBlock(html) {
            if (!html) return false;
            const trimmed = html.trim();
            return /^<table(?:\s[^>]*)?>/i.test(trimmed) && /<\/table>\s*$/i.test(trimmed);
        },

        /**
         * 判断是否是未完成的引用块（有 <blockquote> 但没有 </blockquote>）
         */
        isIncompleteBlockquote(html) {
            if (!html) return false;
            const trimmed = html.trim();
            const hasBlockquoteStart = /<blockquote(?:\s[^>]*)?>/i.test(trimmed);
            const hasBlockquoteEnd = /<\/blockquote>/i.test(trimmed);
            return hasBlockquoteStart && !hasBlockquoteEnd;
        },

        /**
         * 判断是否是引用块内容
         */
        isBlockquoteContent(html) {
            if (!html) return false;
            const trimmed = html.trim();
            // 可以是段落、列表等任何块级元素
            return /^<(p|h[1-6]|ul|ol|blockquote)(?:\s[^>]*)?>/i.test(trimmed);
        },

        /**
         * 将 HTML 按块级元素分割，确保每个块都是完整的 HTML 结构
         */
        splitIntoCompleteBlocks(html) {
            if (!html || !html.trim()) {
                return [];
            }

            const blocks = [];
            const stack = [];
            let currentBlockStart = -1;
            let i = 0;

            const topLevelBlockElements = [
                'p', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
                'ul', 'ol', 'blockquote', 'pre',
                'table', 'hr'
            ];

            while (i < html.length) {
                // 处理开始标签
                if (html[i] === '<' && html[i + 1] !== '/' && html[i + 1] !== '!') {
                    const result = this.processStartTag(html, i, stack, currentBlockStart, blocks, topLevelBlockElements);
                    if (result) {
                        currentBlockStart = result.currentBlockStart;
                        i = result.nextIndex;
                        continue;
                    }
                } else if (html[i] === '<' && html[i + 1] === '/') {
                    // 处理闭合标签
                    const result = this.processCloseTag(html, i, stack, currentBlockStart, blocks, topLevelBlockElements);
                    if (result) {
                        currentBlockStart = result.currentBlockStart;
                    }
                }
                i++;
            }

            // 处理最后一个块
            this.processLastBlock(html, currentBlockStart, blocks);

            return blocks;
        },

        /**
         * 处理开始标签
         */
        processStartTag(html, i, stack, currentBlockStart, blocks, topLevelBlockElements) {
            const tagMatch = html.substring(i).match(/^<(\w+)(?:\s[^>]*)?(?:\s*\/)?>/);
            if (!tagMatch) return null;

            const tagName = tagMatch[1].toLowerCase();
            const isTopLevelBlock = topLevelBlockElements.includes(tagName);
            const isSelfClosing = tagMatch[0].endsWith('/>') || this.isSelfClosingTag(tagName);

            if (!isTopLevelBlock) {
                if (stack.length > 0 && !isSelfClosing) {
                    stack.push({ name: tagName, index: i });
                }
                return null;
            }

            if (isSelfClosing) {
                return this.handleSelfClosingBlock(html, i, tagMatch, currentBlockStart, blocks);
            } else {
                return this.handleNonSelfClosingBlock(html, i, tagName, stack, currentBlockStart, blocks);
            }
        },

        /**
         * 处理自闭合的块级元素
         */
        handleSelfClosingBlock(html, i, tagMatch, currentBlockStart, blocks) {
            if (currentBlockStart >= 0 && i > currentBlockStart) {
                const prevBlock = html.substring(currentBlockStart, i).trim();
                if (prevBlock) {
                    blocks.push(prevBlock);
                }
            }
            const tagEnd = i + tagMatch[0].length;
            blocks.push(html.substring(i, tagEnd));
            return { currentBlockStart: -1, nextIndex: tagEnd };
        },

        /**
         * 处理非自闭合的块级元素
         */
        handleNonSelfClosingBlock(html, i, tagName, stack, currentBlockStart, blocks) {
            if (currentBlockStart >= 0 && i > currentBlockStart) {
                const prevBlock = html.substring(currentBlockStart, i).trim();
                if (prevBlock) {
                    blocks.push(prevBlock);
                }
            }
            stack.push({ name: tagName, index: i });
            return { currentBlockStart: i, nextIndex: i + 1 };
        },

        /**
         * 处理闭合标签
         */
        processCloseTag(html, i, stack, currentBlockStart, blocks, topLevelBlockElements) {
            const closeTagMatch = html.substring(i).match(/^<\/(\w+)>/);
            if (!closeTagMatch || stack.length === 0) return null;

            const tagName = closeTagMatch[1].toLowerCase();
            
            // 查找匹配的开始标签
            for (let j = stack.length - 1; j >= 0; j--) {
                if (stack[j].name === tagName) {
                    stack.splice(j);
                    
                    // 如果栈为空，说明一个完整的顶级块结束了
                    if (stack.length === 0 && currentBlockStart >= 0 && topLevelBlockElements.includes(tagName)) {
                        const blockEnd = i + closeTagMatch[0].length;
                        const completeBlock = html.substring(currentBlockStart, blockEnd);
                        blocks.push(completeBlock);
                        return { currentBlockStart: -1 };
                    }
                    break;
                }
            }
            return null;
        },

        /**
         * 处理最后一个块
         */
        processLastBlock(html, currentBlockStart, blocks) {
            if (currentBlockStart >= 0 && currentBlockStart < html.length) {
                const lastBlock = html.substring(currentBlockStart).trim();
                if (lastBlock) {
                    blocks.push(lastBlock);
                }
            } else if (blocks.length === 0) {
                blocks.push(html);
            }
        },

        /**
         * 判断是否是自闭合标签
         */
        isSelfClosingTag(tagName) {
            const selfClosingTags = ['img', 'br', 'hr', 'input'];
            return selfClosingTags.includes(tagName.toLowerCase());
        },

        /**
         * 检查 HTML 块是否包含图片
         * 用于判断是否可以安全合并块而不导致图片闪烁
         */
        containsImage(html) {
            if (!html) return false;
            // 检查是否包含 <img> 标签
            return /<img[\s\S]*?>/i.test(html);
        }
    }
}
</script>

<style scoped>
.markdown {
    line-height: 1.6;
}

.markdown-block {
    margin-bottom: 0.5em;
}

/* Markdown 样式 */
.markdown :deep(h1),
.markdown :deep(h2),
.markdown :deep(h3),
.markdown :deep(h4),
.markdown :deep(h5),
.markdown :deep(h6) {
    margin-top: 1em;
    margin-bottom: 0.5em;
    font-weight: bold;
}

.markdown :deep(h1) { font-size: 2em; }
.markdown :deep(h2) { font-size: 1.5em; }
.markdown :deep(h3) { font-size: 1.25em; }

.markdown :deep(p) {
    margin: 0.5em 0;
}

.markdown :deep(ul),
.markdown :deep(ol) {
    margin: 0.5em 0;
    padding-left: 2em;
}

.markdown :deep(li) {
    margin: 0.25em 0;
}

.markdown :deep(table) {
    border-collapse: collapse;
    width: 100%;
    margin: 1em 0;
}

.markdown :deep(table.markdown-table) {
    border: 1px solid #ddd;
}

.markdown :deep(th),
.markdown :deep(td) {
    border: 1px solid #ddd;
    padding: 0.5em;
    text-align: left;
}

.markdown :deep(th) {
    background-color: #f5f5f5;
    font-weight: bold;
}

.markdown :deep(img) {
    max-width: 100%;
    height: auto;
    display: block;
    margin: 1em auto;
}

.markdown :deep(code) {
    background-color: #f4f4f4;
    padding: 0.2em 0.4em;
    border-radius: 3px;
    font-family: monospace;
}

.markdown :deep(pre) {
    background-color: #f4f4f4;
    padding: 1em;
    border-radius: 5px;
    overflow-x: auto;
}

.markdown :deep(pre code) {
    background-color: transparent;
    padding: 0;
}

.markdown :deep(blockquote) {
    border-left: 4px solid #ddd;
    padding-left: 1em;
    margin: 1em 0;
    color: #666;
}

.markdown :deep(a) {
    color: #0066cc;
    text-decoration: none;
}

.markdown :deep(a:hover) {
    text-decoration: underline;
}
</style>