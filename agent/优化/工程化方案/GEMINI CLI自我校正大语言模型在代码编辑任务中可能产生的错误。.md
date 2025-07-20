``` TypeScript
/**
 * @license
 * Copyright 2025 Google LLC
 * SPDX-License-Identifier: Apache-2.0
 */

// 导入 GenAI 库中的相关类型
import {
  Content, // 内容块的类型
  GenerateContentConfig, // 生成内容配置的类型
  SchemaUnion, // JSON 模式的联合类型
  Type, // JSON 模式中类型枚举
} from '@google/genai';
// 导入 GeminiClient，用于与 Gemini 模型进行交互
import { GeminiClient } from '../core/client.js';
// 导入 EditToolParams，这是编辑工具的参数类型，可能来自另一个文件
import { EditToolParams } from '../tools/edit.js';
// 导入 LruCache，一个简单的 LRU 缓存实现
import { LruCache } from './LruCache.js';
// 导入默认的 Gemini Flash 模型名称
import { DEFAULT_GEMINI_FLASH_MODEL } from '../config/models.js';

// 定义用于编辑校正的 Gemini 模型
const EditModel = DEFAULT_GEMINI_FLASH_MODEL;
// 定义用于编辑校正的生成配置，thinkingBudget 设置为 0 表示不进行额外的思考预算
const EditConfig: GenerateContentConfig = {
  thinkingConfig: {
    thinkingBudget: 0,
  },
};

// 定义缓存的最大大小
const MAX_CACHE_SIZE = 50;

// 用于存储 ensureCorrectEdit 结果的 LRU 缓存
const editCorrectionCache = new LruCache<string, CorrectedEditResult>(
  MAX_CACHE_SIZE,
);

// 用于存储 ensureCorrectFileContent 结果的 LRU 缓存
const fileContentCorrectionCache = new LruCache<string, string>(MAX_CACHE_SIZE);

/**
 * 定义 CorrectedEditResult 中参数的结构。
 * 这些是经过校正后的编辑操作的参数。
 */
interface CorrectedEditParams {
  file_path: string; // 文件路径
  old_string: string; // 要替换的旧字符串
  new_string: string; // 替换后的新字符串
}

/**
 * 定义 ensureCorrectEdit 函数的结果结构。
 */
export interface CorrectedEditResult {
  params: CorrectedEditParams; // 校正后的编辑参数
  occurrences: number; // old_string 在文件中出现的次数
}

/**
 * 尝试校正编辑参数，如果原始的 old_string 未找到。
 * 它会尝试取消转义，然后进行基于 LLM 的校正。
 * 结果会被缓存以避免重复处理。
 *
 * @param currentContent 文件的当前内容。
 * @param originalParams 原始的 EditToolParams（来自 edit.ts，未经过校正）。
 * @param client 用于 LLM 调用的 GeminiClient 实例。
 * @param abortSignal 用于中止 LLM 调用的 AbortSignal。
 * @returns 一个 Promise，解析为一个对象，包含（可能已校正的）EditToolParams
 * （作为 CorrectedEditParams）和最终的出现次数。
 */
export async function ensureCorrectEdit(
  currentContent: string,
  originalParams: EditToolParams, // 这是来自 edit.ts 的 EditToolParams，没有 'corrected' 字段
  client: GeminiClient,
  abortSignal: AbortSignal,
): Promise<CorrectedEditResult> {
  // 构建缓存键
  const cacheKey = `${currentContent}---${originalParams.old_string}---${originalParams.new_string}`;
  // 尝试从缓存中获取结果
  const cachedResult = editCorrectionCache.get(cacheKey);
  if (cachedResult) {
    return cachedResult; // 如果缓存命中，则直接返回缓存结果
  }

  let finalNewString = originalParams.new_string; // 最终的新字符串，初始为原始新字符串
  // 检查新字符串是否可能被转义（通过尝试取消转义并比较）
  const newStringPotentiallyEscaped =
    unescapeStringForGeminiBug(originalParams.new_string) !==
    originalParams.new_string;

  // 期望的替换次数，默认为 1
  const expectedReplacements = originalParams.expected_replacements ?? 1;

  let finalOldString = originalParams.old_string; // 最终的旧字符串，初始为原始旧字符串
  let occurrences = countOccurrences(currentContent, finalOldString); // 计算原始旧字符串的出现次数

  // 情况 1: 原始 old_string 出现次数与期望的替换次数匹配
  if (occurrences === expectedReplacements) {
    // 如果新字符串可能被转义，则校正新字符串的转义
    if (newStringPotentiallyEscaped) {
      finalNewString = await correctNewStringEscaping(
        client,
        finalOldString,
        originalParams.new_string,
        abortSignal,
      );
    }
  }
  // 情况 2: 原始 old_string 出现次数大于期望的替换次数
  else if (occurrences > expectedReplacements) {
    // 如果用户期望多次替换，直接返回（不做校正，后续验证会失败）
    if (occurrences === expectedReplacements) {
      const result: CorrectedEditResult = {
        params: { ...originalParams },
        occurrences,
      };
      editCorrectionCache.set(cacheKey, result);
      return result;
    }

    // 如果用户期望 1 次但找到多次，尝试校正（现有行为）
    if (expectedReplacements === 1) {
      const result: CorrectedEditResult = {
        params: { ...originalParams },
        occurrences,
      };
      editCorrectionCache.set(cacheKey, result);
      return result;
    }

    // 如果出现次数与期望不匹配，直接返回（后续验证会失败）
    const result: CorrectedEditResult = {
      params: { ...originalParams },
      occurrences,
    };
    editCorrectionCache.set(cacheKey, result);
    return result;
  }
  // 情况 3: 原始 old_string 出现次数为 0 或其他意外状态
  else {
    // 尝试对 old_string 进行取消转义
    const unescapedOldStringAttempt = unescapeStringForGeminiBug(
      originalParams.old_string,
    );
    occurrences = countOccurrences(currentContent, unescapedOldStringAttempt); // 重新计算出现次数

    // 如果取消转义后的 old_string 出现次数与期望的替换次数匹配
    if (occurrences === expectedReplacements) {
      finalOldString = unescapedOldStringAttempt; // 使用取消转义后的 old_string
      // 如果新字符串可能被转义，则校正新字符串
      if (newStringPotentiallyEscaped) {
        finalNewString = await correctNewString(
          client,
          originalParams.old_string, // 原始旧字符串
          unescapedOldStringAttempt, // 校正后的旧字符串
          originalParams.new_string, // 原始新字符串（可能被转义）
          abortSignal,
        );
      }
    }
    // 如果取消转义后 old_string 出现次数仍为 0
    else if (occurrences === 0) {
      // 尝试使用 LLM 校正 old_string
      const llmCorrectedOldString = await correctOldStringMismatch(
        client,
        currentContent,
        unescapedOldStringAttempt,
        abortSignal,
      );
      // 计算 LLM 校正后的 old_string 的出现次数
      const llmOldOccurrences = countOccurrences(
        currentContent,
        llmCorrectedOldString,
      );

      // 如果 LLM 校正后的 old_string 出现次数与期望的替换次数匹配
      if (llmOldOccurrences === expectedReplacements) {
        finalOldString = llmCorrectedOldString; // 使用 LLM 校正后的 old_string
        occurrences = llmOldOccurrences; // 更新出现次数

        // 如果新字符串可能被转义，则校正新字符串
        if (newStringPotentiallyEscaped) {
          const baseNewStringForLLMCorrection = unescapeStringForGeminiBug(
            originalParams.new_string,
          );
          finalNewString = await correctNewString(
            client,
            originalParams.old_string, // 原始旧字符串
            llmCorrectedOldString, // 校正后的旧字符串
            baseNewStringForLLMCorrection, // 用于校正的新字符串基础（已取消转义）
            abortSignal,
          );
        }
      } else {
        // LLM 校正 old_string 也失败了
        const result: CorrectedEditResult = {
          params: { ...originalParams },
          occurrences: 0, // 明确设置为 0，因为 LLM 失败
        };
        editCorrectionCache.set(cacheKey, result);
        return result;
      }
    } else {
      // 取消转义 old_string 导致出现次数 > 1
      const result: CorrectedEditResult = {
        params: { ...originalParams },
        occurrences, // 这将是 > 1
      };
      editCorrectionCache.set(cacheKey, result);
      return result;
    }
  }

  // 尝试修剪 old_string 和 new_string 的空白，如果修剪后的 old_string 仍然匹配期望的出现次数
  const { targetString, pair } = trimPairIfPossible(
    finalOldString,
    finalNewString,
    currentContent,
    expectedReplacements,
  );
  finalOldString = targetString;
  finalNewString = pair;

  // 构建最终结果
  const result: CorrectedEditResult = {
    params: {
      file_path: originalParams.file_path,
      old_string: finalOldString,
      new_string: finalNewString,
    },
    occurrences: countOccurrences(currentContent, finalOldString), // 使用最终的 old_string 重新计算出现次数
  };
  editCorrectionCache.set(cacheKey, result); // 缓存结果
  return result;
}

/**
 * 尝试校正文件内容中的转义问题。
 * 如果内容可能被转义，则使用 LLM 进行校正。
 * 结果会被缓存以避免重复处理。
 *
 * @param content 文件的内容。
 * @param client 用于 LLM 调用的 GeminiClient 实例。
 * @param abortSignal 用于中止 LLM 调用的 AbortSignal。
 * @returns 一个 Promise，解析为校正后的文件内容。
 */
export async function ensureCorrectFileContent(
  content: string,
  client: GeminiClient,
  abortSignal: AbortSignal,
): Promise<string> {
  // 尝试从缓存中获取结果
  const cachedResult = fileContentCorrectionCache.get(content);
  if (cachedResult) {
    return cachedResult; // 如果缓存命中，则直接返回缓存结果
  }

  // 检查内容是否可能被转义
  const contentPotentiallyEscaped =
    unescapeStringForGeminiBug(content) !== content;
  if (!contentPotentiallyEscaped) {
    fileContentCorrectionCache.set(content, content); // 如果没有转义，则缓存原始内容并返回
    return content;
  }

  // 使用 LLM 校正字符串转义
  const correctedContent = await correctStringEscaping(
    content,
    client,
    abortSignal,
  );
  fileContentCorrectionCache.set(content, correctedContent); // 缓存校正后的内容
  return correctedContent;
}

// 定义用于 LLM 响应中 old_string 校正的预期 JSON 模式
const OLD_STRING_CORRECTION_SCHEMA: SchemaUnion = {
  type: Type.OBJECT,
  properties: {
    corrected_target_snippet: {
      type: Type.STRING,
      description:
        'The corrected version of the target snippet that exactly and uniquely matches a segment within the provided file content.',
    },
  },
  required: ['corrected_target_snippet'], // 必须包含此属性
};

/**
 * 校正 old_string 不匹配的问题。
 * 当原始 old_string 未在文件内容中找到时，使用 LLM 尝试找到最匹配的片段。
 *
 * @param geminiClient GeminiClient 实例。
 * @param fileContent 文件的当前内容。
 * @param problematicSnippet 无法匹配的旧字符串片段。
 * @param abortSignal 用于中止 LLM 调用的 AbortSignal。
 * @returns 一个 Promise，解析为校正后的旧字符串片段。
 */
export async function correctOldStringMismatch(
  geminiClient: GeminiClient,
  fileContent: string,
  problematicSnippet: string,
  abortSignal: AbortSignal,
): Promise<string> {
  // 构建 LLM 提示
  const prompt = `
Context: A process needs to find an exact literal, unique match for a specific text snippet within a file's content. The provided snippet failed to match exactly. This is most likely because it has been overly escaped.

Task: Analyze the provided file content and the problematic target snippet. Identify the segment in the file content that the snippet was *most likely* intended to match. Output the *exact*, literal text of that segment from the file content. Focus *only* on removing extra escape characters and correcting formatting, whitespace, or minor differences to achieve a PERFECT literal match. The output must be the exact literal text as it appears in the file.

Problematic target snippet:
\`\`\`
${problematicSnippet}
\`\`\`

File Content:
\`\`\`
${fileContent}
\`\`\`

For example, if the problematic target snippet was "\\\\\\nconst greeting = \`Hello \\\\\`\${name}\\\\\`\`;" and the file content had content that looked like "\nconst greeting = \`Hello ${'\\`'}\${name}${'\\`'}\`;", then corrected_target_snippet should likely be "\nconst greeting = \`Hello ${'\\`'}\${name}${'\\`'}\`;" to fix the incorrect escaping to match the original file content.
If the differences are only in whitespace or formatting, apply similar whitespace/formatting changes to the corrected_target_snippet.

Return ONLY the corrected target snippet in the specified JSON format with the key 'corrected_target_snippet'. If no clear, unique match can be found, return an empty string for 'corrected_target_snippet'.
`.trim();

  // 构建 LLM 请求内容
  const contents: Content[] = [{ role: 'user', parts: [{ text: prompt }] }];

  try {
    // 调用 Gemini LLM 生成 JSON 响应
    const result = await geminiClient.generateJson(
      contents,
      OLD_STRING_CORRECTION_SCHEMA,
      abortSignal,
      EditModel,
      EditConfig,
    );

    // 检查结果是否有效并返回校正后的片段
    if (
      result &&
      typeof result.corrected_target_snippet === 'string' &&
      result.corrected_target_snippet.length > 0
    ) {
      return result.corrected_target_snippet;
    } else {
      return problematicSnippet; // 如果 LLM 未能校正，返回原始片段
    }
  } catch (error) {
    if (abortSignal.aborted) {
      throw error; // 如果是中止信号导致的错误，则重新抛出
    }

    console.error(
      'Error during LLM call for old string snippet correction:',
      error,
    );

    return problematicSnippet; // 发生其他错误时，返回原始片段
  }
}

// 定义用于 LLM 响应中 new_string 校正的预期 JSON 模式
const NEW_STRING_CORRECTION_SCHEMA: SchemaUnion = {
  type: Type.OBJECT,
  properties: {
    corrected_new_string: {
      type: Type.STRING,
      description:
        'The original_new_string adjusted to be a suitable replacement for the corrected_old_string, while maintaining the original intent of the change.',
    },
  },
  required: ['corrected_new_string'], // 必须包含此属性
};

/**
 * 调整 new_string 以与校正后的 old_string 对齐，同时保持原始意图。
 * 当 old_string 被校正后，new_string 也可能需要相应调整，以确保替换的语义正确。
 *
 * @param geminiClient GeminiClient 实例。
 * @param originalOldString 原始的旧字符串。
 * @param correctedOldString 校正后的旧字符串（实际在文件中找到的）。
 * @param originalNewString 原始的新字符串。
 * @param abortSignal 用于中止 LLM 调用的 AbortSignal。
 * @returns 一个 Promise，解析为校正后的新字符串。
 */
export async function correctNewString(
  geminiClient: GeminiClient,
  originalOldString: string,
  correctedOldString: string,
  originalNewString: string,
  abortSignal: AbortSignal,
): Promise<string> {
  // 如果原始旧字符串和校正后的旧字符串相同，则不需要校正新字符串
  if (originalOldString === correctedOldString) {
    return originalNewString;
  }

  // 构建 LLM 提示
  const prompt = `
Context: A text replacement operation was planned. The original text to be replaced (original_old_string) was slightly different from the actual text in the file (corrected_old_string). The original_old_string has now been corrected to match the file content.
We now need to adjust the replacement text (original_new_string) so that it makes sense as a replacement for the corrected_old_string, while preserving the original intent of the change.

original_old_string (what was initially intended to be found):
\`\`\`
${originalOldString}
\`\`\`

corrected_old_string (what was actually found in the file and will be replaced):
\`\`\`
${correctedOldString}
\`\`\`

original_new_string (what was intended to replace original_old_string):
\`\`\`
${originalNewString}
\`\`\`

Task: Based on the differences between original_old_string and corrected_old_string, and the content of original_new_string, generate a corrected_new_string. This corrected_new_string should be what original_new_string would have been if it was designed to replace corrected_old_string directly, while maintaining the spirit of the original transformation.

For example, if original_old_string was "\\\\\\nconst greeting = \`Hello \\\\\`\${name}\\\\\`\`;" and corrected_old_string is "\nconst greeting = \`Hello ${'\\`'}\${name}${'\\`'}\`;", and original_new_string was "\\\\\\nconst greeting = \`Hello \\\\\`\${name} \${lastName}\\\\\`\`;", then corrected_new_string should likely be "\nconst greeting = \`Hello ${'\\`'}\${name} \${lastName}${'\\`'}\`;" to fix the incorrect escaping.
If the differences are only in whitespace or formatting, apply similar whitespace/formatting changes to the corrected_new_string.

Return ONLY the corrected string in the specified JSON format with the key 'corrected_new_string'. If no adjustment is deemed necessary or possible, return the original_new_string.
  `.trim();

  // 构建 LLM 请求内容
  const contents: Content[] = [{ role: 'user', parts: [{ text: prompt }] }];

  try {
    // 调用 Gemini LLM 生成 JSON 响应
    const result = await geminiClient.generateJson(
      contents,
      NEW_STRING_CORRECTION_SCHEMA,
      abortSignal,
      EditModel,
      EditConfig,
    );

    // 检查结果是否有效并返回校正后的新字符串
    if (
      result &&
      typeof result.corrected_new_string === 'string' &&
      result.corrected_new_string.length > 0
    ) {
      return result.corrected_new_string;
    } else {
      return originalNewString; // 如果 LLM 未能校正，返回原始新字符串
    }
  } catch (error) {
    if (abortSignal.aborted) {
      throw error; // 如果是中止信号导致的错误，则重新抛出
    }

    console.error('Error during LLM call for new_string correction:', error);
    return originalNewString; // 发生其他错误时，返回原始新字符串
  }
}

// 定义用于 LLM 响应中新字符串转义校正的 JSON 模式
const CORRECT_NEW_STRING_ESCAPING_SCHEMA: SchemaUnion = {
  type: Type.OBJECT,
  properties: {
    corrected_new_string_escaping: {
      type: Type.STRING,
      description:
        'The new_string with corrected escaping, ensuring it is a proper replacement for the old_string, especially considering potential over-escaping issues from previous LLM generations.',
    },
  },
  required: ['corrected_new_string_escaping'], // 必须包含此属性
};

/**
 * 校正 new_string 中的转义问题。
 * 专门用于处理 LLM 生成的字符串中可能出现的过度转义问题。
 *
 * @param geminiClient GeminiClient 实例。
 * @param oldString 要替换的旧字符串。
 * @param potentiallyProblematicNewString 可能是问题的新字符串（可能包含不正确的转义）。
 * @param abortSignal 用于中止 LLM 调用的 AbortSignal。
 * @returns 一个 Promise，解析为转义校正后的新字符串。
 */
export async function correctNewStringEscaping(
  geminiClient: GeminiClient,
  oldString: string,
  potentiallyProblematicNewString: string,
  abortSignal: AbortSignal,
): Promise<string> {
  // 构建 LLM 提示
  const prompt = `
Context: A text replacement operation is planned. The text to be replaced (old_string) has been correctly identified in the file. However, the replacement text (new_string) might have been improperly escaped by a previous LLM generation (e.g. too many backslashes for newlines like \\n instead of \n, or unnecessarily quotes like \\"Hello\\" instead of "Hello").

old_string (this is the exact text that will be replaced):
\`\`\`
${oldString}
\`\`\`

potentially_problematic_new_string (this is the text that should replace old_string, but MIGHT have bad escaping, or might be entirely correct):
\`\`\`
${potentiallyProblematicNewString}
\`\`\`

Task: Analyze the potentially_problematic_new_string. If it's syntactically invalid due to incorrect escaping (e.g., "\n", "\t", "\\", "\\'", "\\""), correct the invalid syntax. The goal is to ensure the new_string, when inserted into the code, will be a valid and correctly interpreted.

For example, if old_string is "foo" and potentially_problematic_new_string is "bar\\nbaz", the corrected_new_string_escaping should be "bar\nbaz".
If potentially_problematic_new_string is console.log(\\"Hello World\\"), it should be console.log("Hello World").

Return ONLY the corrected string in the specified JSON format with the key 'corrected_new_string_escaping'. If no escaping correction is needed, return the original potentially_problematic_new_string.
  `.trim();

  // 构建 LLM 请求内容
  const contents: Content[] = [{ role: 'user', parts: [{ text: prompt }] }];

  try {
    // 调用 Gemini LLM 生成 JSON 响应
    const result = await geminiClient.generateJson(
      contents,
      CORRECT_NEW_STRING_ESCAPING_SCHEMA,
      abortSignal,
      EditModel,
      EditConfig,
    );

    // 检查结果是否有效并返回校正后的新字符串
    if (
      result &&
      typeof result.corrected_new_string_escaping === 'string' &&
      result.corrected_new_string_escaping.length > 0
    ) {
      return result.corrected_new_string_escaping;
    } else {
      return potentiallyProblematicNewString; // 如果 LLM 未能校正，返回原始新字符串
    }
  } catch (error) {
    if (abortSignal.aborted) {
      throw error; // 如果是中止信号导致的错误，则重新抛出
    }

    console.error(
      'Error during LLM call for new_string escaping correction:',
      error,
    );
    return potentiallyProblematicNewString; // 发生其他错误时，返回原始新字符串
  }
}

// 定义用于 LLM 响应中字符串转义校正的 JSON 模式
const CORRECT_STRING_ESCAPING_SCHEMA: SchemaUnion = {
  type: Type.OBJECT,
  properties: {
    corrected_string_escaping: {
      type: Type.STRING,
      description:
        'The string with corrected escaping, ensuring it is valid, specially considering potential over-escaping issues from previous LLM generations.',
    },
  },
  required: ['corrected_string_escaping'], // 必须包含此属性
};

/**
 * 校正通用字符串中的转义问题。
 * 用于校正 LLM 生成的任何可能包含过度转义的字符串。
 *
 * @param potentiallyProblematicString 可能是问题的字符串（可能包含不正确的转义）。
 * @param client 用于 LLM 调用的 GeminiClient 实例。
 * @param abortSignal 用于中止 LLM 调用的 AbortSignal。
 * @returns 一个 Promise，解析为转义校正后的字符串。
 */
export async function correctStringEscaping(
  potentiallyProblematicString: string,
  client: GeminiClient,
  abortSignal: AbortSignal,
): Promise<string> {
  // 构建 LLM 提示
  const prompt = `
Context: An LLM has just generated potentially_problematic_string and the text might have been improperly escaped (e.g. too many backslashes for newlines like \\n instead of \n, or unnecessarily quotes like \\"Hello\\" instead of "Hello").

potentially_problematic_string (this text MIGHT have bad escaping, or might be entirely correct):
\`\`\`
${potentiallyProblematicString}
\`\`\`

Task: Analyze the potentially_problematic_string. If it's syntactically invalid due to incorrect escaping (e.g., "\n", "\t", "\\", "\\'", "\\""), correct the invalid syntax. The goal is to ensure the text will be a valid and correctly interpreted.

For example, if potentially_problematic_string is "bar\\nbaz", the corrected_new_string_escaping should be "bar\nbaz".
If potentially_problematic_string is console.log(\\"Hello World\\"), it should be console.log("Hello World").

Return ONLY the corrected string in the specified JSON format with the key 'corrected_string_escaping'. If no escaping correction is needed, return the original potentially_problematic_string.
  `.trim();

  // 构建 LLM 请求内容
  const contents: Content[] = [{ role: 'user', parts: [{ text: prompt }] }];

  try {
    // 调用 Gemini LLM 生成 JSON 响应
    const result = await client.generateJson(
      contents,
      CORRECT_STRING_ESCAPING_SCHEMA,
      abortSignal,
      EditModel,
      EditConfig,
    );

    // 检查结果是否有效并返回校正后的字符串
    if (
      result &&
      typeof result.corrected_string_escaping === 'string' &&
      result.corrected_string_escaping.length > 0
    ) {
      return result.corrected_string_escaping;
    } else {
      return potentiallyProblematicString; // 如果 LLM 未能校正，返回原始字符串
    }
  } catch (error) {
    if (abortSignal.aborted) {
      throw error; // 如果是中止信号导致的错误，则重新抛出
    }

    console.error(
      'Error during LLM call for string escaping correction:',
      error,
    );
    return potentiallyProblematicString; // 发生其他错误时，返回原始字符串
  }
}

/**
 * 尽可能修剪字符串对（target 和 pair）的空白。
 * 如果修剪 target 后其出现次数仍与期望的替换次数匹配，则同时修剪 pair。
 *
 * @param target 目标字符串（通常是 old_string）。
 * @param trimIfTargetTrims 如果 target 被修剪，则也修剪此字符串（通常是 new_string）。
 * @param currentContent 文件的当前内容。
 * @param expectedReplacements 期望的替换次数。
 * @returns 包含修剪后的 targetString 和 pair 的对象。
 */
function trimPairIfPossible(
  target: string,
  trimIfTargetTrims: string,
  currentContent: string,
  expectedReplacements: number,
) {
  const trimmedTargetString = target.trim(); // 修剪目标字符串
  // 如果修剪后的长度与原始长度不同（即有空白被修剪）
  if (target.length !== trimmedTargetString.length) {
    // 计算修剪后的目标字符串在文件中的出现次数
    const trimmedTargetOccurrences = countOccurrences(
      currentContent,
      trimmedTargetString,
    );

    // 如果修剪后的目标字符串的出现次数与期望的替换次数匹配
    if (trimmedTargetOccurrences === expectedReplacements) {
      const trimmedReactiveString = trimIfTargetTrims.trim(); // 修剪配对字符串
      return {
        targetString: trimmedTargetString, // 返回修剪后的目标字符串
        pair: trimmedReactiveString, // 返回修剪后的配对字符串
      };
    }
  }

  // 如果没有修剪或修剪后不匹配，则返回原始字符串
  return {
    targetString: target,
    pair: trimIfTargetTrims,
  };
}

/**
 * 取消转义可能被 LLM 过度转义的字符串。
 * 例如，将 "\\n" 转换为 "\n"，将 "\\\\" 转换为 "\"。
 * @param inputString 要取消转义的输入字符串。
 * @returns 取消转义后的字符串。
 */
export function unescapeStringForGeminiBug(inputString: string): string {
  // 正则表达式解释：
  // \\ : 匹配一个字面反斜杠字符。
  // (n|t|r|'|"|`|\\|\n) : 这是一个捕获组。它匹配以下之一：
  //   n, t, r, ', ", ` : 这些匹配字面字符 'n', 't', 'r', 单引号, 双引号, 或反引号。
  //                      这处理了像 "\\n", "\\`" 等情况。
  //   \\ : 这匹配一个字面反斜杠。这处理了像 "\\\\" (转义的反斜杠) 等情况。
  //   \n : 这匹配一个实际的换行符。这处理了输入字符串中可能出现像 "\\\n"
  //        (一个字面反斜杠后跟一个换行符) 的情况。
  // g : 全局标志，替换所有匹配项。

  return inputString.replace(
    /\\+(n|t|r|'|"|`|\\|\n)/g, // 匹配一个或多个反斜杠后跟一个特殊字符或实际换行符
    (match, capturedChar) => {
      // 'match' 是整个错误的序列，例如，如果输入（内存中）是 "\\\\`"，则 match 是 "\\\\`"。
      // 'capturedChar' 是决定真实含义的字符，例如 '`'。

      switch (capturedChar) {
        case 'n':
          return '\n'; // 正确转义：\n (换行符)
        case 't':
          return '\t'; // 正确转义：\t (制表符)
        case 'r':
          return '\r'; // 正确转义：\r (回车符)
        case "'":
          return "'"; // 正确转义：' (撇号字符)
        case '"':
          return '"'; // 正确转义：" (引号字符)
        case '`':
          return '`'; // 正确转义：` (反引号字符)
        case '\\': // 这处理了当 'capturedChar' 是字面反斜杠时的情况
          return '\\'; // 将转义的反斜杠（例如，"\\\\"）替换为单个反斜杠
        case '\n': // 这处理了当 'capturedChar' 是实际换行符时的情况
          return '\n'; // 将整个错误的序列（例如，内存中的 "\\\n"）替换为干净的换行符
        default:
          // 理想情况下，如果正则表达式捕获正确，则不应到达此回退。
          // 如果捕获到意外字符，它将返回原始匹配序列。
          return match;
      }
    },
  );
}

/**
 * 统计子字符串在字符串中出现的次数。
 * @param str 要搜索的字符串。
 * @param substr 要查找的子字符串。
 * @returns 子字符串在字符串中出现的次数。
 */
export function countOccurrences(str: string, substr: string): number {
  if (substr === '') {
    return 0; // 如果子字符串为空，则出现次数为 0
  }
  let count = 0;
  let pos = str.indexOf(substr); // 查找第一次出现的位置
  while (pos !== -1) {
    count++; // 增加计数
    pos = str.indexOf(substr, pos + substr.length); // 从当前匹配的末尾之后开始下一次搜索
  }
  return count; // 返回总计数
}

/**
 * 仅用于测试：重置编辑校正器缓存。
 */
export function resetEditCorrectorCaches_TEST_ONLY() {
  editCorrectionCache.clear(); // 清空编辑校正缓存
  fileContentCorrectionCache.clear(); // 清空文件内容校正缓存
}

```