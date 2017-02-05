```java
import java.util.ArrayList;
import java.util.List;

public class Solution {
  private static class Entry {
    boolean any;
    boolean asterisk;
    char ch;

    boolean matchChar(char c) {
      return any || c == ch;
    }
  }

  public boolean isMatch(String str, String p) {
    char[] pattern = p.toCharArray();
    List<Entry> entries = new ArrayList<>();
    for (int index = 0; index < pattern.length; index++) {
      Entry entry = new Entry();
      entry.ch = pattern[index];
      if (pattern[index] == '.') {
        entry.any = true;
      }
      if (index + 1 < pattern.length && pattern[index + 1] == '*') {
        entry.asterisk = true;
        index++;
      }
      entries.add(entry);
    }
    int rows = str.length() + 1;
    int cols = entries.size() + 1;
    // matrix[i][j]的含义为: 前i个字符和前j个entry是否匹配
    boolean[][] matrix = new boolean[rows][cols];

    // 设定左上角 0个字符匹配0个entry, 所以设置为true
    matrix[0][0] = true;
    // 设定第一行
    // 0个字符匹配连续n个带星号的entry
    // 但只要出现一个不带星号的entry, 后续的entry就都为不匹配
    for (int j = 1; j < cols; j++) {
      if (entries.get(j - 1).asterisk) {
        matrix[0][j] = true;
      } else {
        break;
      }
    }
    // 第一列均为false, 就不需要额外的设置了

    // 开始计算matrix中的每一项
    for (int i = 1; i < rows; i++) {
      for (int j = 1; j < cols; j++) {
        char c = str.charAt(i - 1);
        Entry entry = entries.get(j - 1);

        // 选项1 单个字符匹配单个entry
        boolean option1 = matrix[i - 1][j - 1] && entry.matchChar(c);
        // 选项2 单个字符匹配上一个entry的星号
        boolean option2 = matrix[i - 1][j] && entry.matchChar(c) && entry.asterisk;
        // 选项3 entry带星号, 不匹配任意字符
        boolean option3 = matrix[i][j - 1] && entry.asterisk;

        matrix[i][j] = option1 || option2 || option3;
      }
    }

    return matrix[rows - 1][cols - 1];
  }
}
```