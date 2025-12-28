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
            blockCounter: 0
        };
    },
    watch: {
        content: {
            immediate: true,
            handler(newContent, oldContent) {
                this.handleContentUpdate(newContent);
            }
        }
    },
    methods: {
        /**
         * 处理内容更新，使用增量渲染策略
         */
        handleContentUpdate(newContent) {
            if (!newContent) {
                this.renderBlocks = [];
                this.processedLength = 0;
                this.blockCounter = 0;
                return;
            }

            // 如果内容长度减少，说明是重置，重新渲染
            if (newContent.length < this.processedLength) {
                this.renderBlocks = [];
                this.processedLength = 0;
                this.blockCounter = 0;
            }

            // 获取新增的内容部分
            const newPart = newContent.substring(this.processedLength);
            
            if (newPart.trim()) {
                // 增量渲染新内容
                this.appendNewContent(newPart);
            }

            this.processedLength = newContent.length;
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
                
                // 智能合并未完成的段落
                blocks.forEach((blockHtml) => {
                    if (blockHtml.trim()) {
                        // 检查最后一个块是否是未完成的段落
                        const lastBlock = this.renderBlocks[this.renderBlocks.length - 1];
                        const isLastBlockParagraph = lastBlock && this.isParagraphBlock(lastBlock.html);
                        const isCurrentBlockParagraph = this.isParagraphBlock(blockHtml);
                        
                        // 如果最后一个块是段落，且当前块也是段落，且最后一个块未完成，则合并它们
                        if (isLastBlockParagraph && isCurrentBlockParagraph && !this.isParagraphComplete(lastBlock.html)) {
                            // 合并段落：移除最后一个块的 </p>，移除当前块的 <p>，合并内容
                            const lastBlockContent = lastBlock.html.replace(/<\/p>\s*$/, '');
                            const currentBlockContent = blockHtml.replace(/^<p[^>]*>/, '').replace(/<\/p>\s*$/, '');
                            lastBlock.html = lastBlockContent + currentBlockContent + '</p>';
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
         * 判断一个 HTML 块是否是段落（<p> 标签）
         */
        isParagraphBlock(html) {
            if (!html) return false;
            // 检查是否以 <p> 开始并以 </p> 结束
            const trimmed = html.trim();
            return /^<p(?:\s[^>]*)?>/.test(trimmed) && /<\/p>\s*$/.test(trimmed);
        },

        /**
         * 判断段落是否已完成（以句号、问号、感叹号等结束）
         * 如果段落已完成，不应该与下一个段落合并
         */
        isParagraphComplete(html) {
            if (!html) return false;
            // 提取段落内的文本内容（去除 HTML 标签）
            const textContent = html.replace(/<[^>]+>/g, '').trim();
            // 检查是否以句号、问号、感叹号、换行符等结束
            // 也检查是否包含图片（图片通常表示段落结束）
            return /[。！？.!?]\s*$/.test(textContent) || /<img/.test(html);
        },

        /**
         * 将 HTML 按块级元素分割，确保每个块都是完整的 HTML 结构
         * 使用栈来正确跟踪嵌套的 HTML 标签，确保只在顶级块边界分割
         */
        splitIntoCompleteBlocks(html) {
            if (!html || !html.trim()) {
                return [];
            }

            const blocks = [];
            const stack = []; // 用于跟踪嵌套的标签
            let currentBlockStart = -1;
            let i = 0;

            // 顶级块级元素（这些元素通常不会嵌套在其他块级元素中，或者嵌套时应该作为一个整体）
            const topLevelBlockElements = [
                'p', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
                'ul', 'ol', 'blockquote', 'pre',
                'table', 'hr'
            ];

            while (i < html.length) {
                // 查找开始标签
                if (html[i] === '<' && html[i + 1] !== '/' && html[i + 1] !== '!') {
                    const tagMatch = html.substring(i).match(/^<(\w+)(?:\s[^>]*)?(?:\s*\/)?>/);
                    if (tagMatch) {
                        const tagName = tagMatch[1].toLowerCase();
                        const isTopLevelBlock = topLevelBlockElements.includes(tagName);
                        const isSelfClosing = tagMatch[0].endsWith('/>') || this.isSelfClosingTag(tagName);

                        if (isTopLevelBlock) {
                            if (isSelfClosing) {
                                // 自闭合的块级元素（如 <hr />）
                                if (stack.length === 0) {
                                    // 保存之前的块（如果有）
                                    if (currentBlockStart >= 0 && i > currentBlockStart) {
                                        const prevBlock = html.substring(currentBlockStart, i).trim();
                                        if (prevBlock) {
                                            blocks.push(prevBlock);
                                        }
                                    }
                                    // 添加自闭合标签作为一个块
                                    const tagEnd = i + tagMatch[0].length;
                                    blocks.push(html.substring(i, tagEnd));
                                    currentBlockStart = -1;
                                    i = tagEnd;
                                    continue;
                                }
                            } else {
                                // 非自闭合的块级元素
                                if (stack.length === 0) {
                                    // 这是一个新的顶级块
                                    if (currentBlockStart >= 0 && i > currentBlockStart) {
                                        const prevBlock = html.substring(currentBlockStart, i).trim();
                                        if (prevBlock) {
                                            blocks.push(prevBlock);
                                        }
                                    }
                                    currentBlockStart = i;
                                }
                                stack.push({ name: tagName, index: i });
                            }
                        } else if (stack.length > 0 && !isSelfClosing) {
                            // 在块内的标签，继续跟踪
                            stack.push({ name: tagName, index: i });
                        }
                    }
                } else if (html[i] === '<' && html[i + 1] === '/') {
                    // 闭合标签
                    const closeTagMatch = html.substring(i).match(/^<\/(\w+)>/);
                    if (closeTagMatch && stack.length > 0) {
                        const tagName = closeTagMatch[1].toLowerCase();
                        
                        // 查找匹配的开始标签
                        for (let j = stack.length - 1; j >= 0; j--) {
                            if (stack[j].name === tagName) {
                                // 移除从 j 到栈顶的所有标签
                                stack.splice(j);
                                
                                // 如果栈为空，说明一个完整的顶级块结束了
                                if (stack.length === 0 && currentBlockStart >= 0 && topLevelBlockElements.includes(tagName)) {
                                    const blockEnd = i + closeTagMatch[0].length;
                                    const completeBlock = html.substring(currentBlockStart, blockEnd);
                                    blocks.push(completeBlock);
                                    currentBlockStart = -1;
                                }
                                break;
                            }
                        }
                    }
                }
                i++;
            }

            // 处理最后一个块（如果有）
            if (currentBlockStart >= 0 && currentBlockStart < html.length) {
                const lastBlock = html.substring(currentBlockStart).trim();
                if (lastBlock) {
                    blocks.push(lastBlock);
                }
            } else if (blocks.length === 0) {
                // 如果没有找到任何块级元素，返回整个 HTML
                blocks.push(html);
            }

            return blocks;
        },

        /**
         * 判断是否是自闭合标签
         */
        isSelfClosingTag(tagName) {
            const selfClosingTags = ['img', 'br', 'hr', 'input'];
            return selfClosingTags.includes(tagName.toLowerCase());
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