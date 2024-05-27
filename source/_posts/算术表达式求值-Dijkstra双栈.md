---
title: 算术表达式求值-Dijkstra双栈
tags: [算法，算术表达式，Dijkstra双栈]
index_img: /img/Algorithm-Image-1.png
date: 2024-05-27 22:00:00
---

双栈结构用于解决一些特定类型的问题，如括号匹配、逆波兰表达式求值等。在数学公式计算中，我们可以使用两个栈来存储操作数和运算符。

以下是一个简单的Java实现：

```java
public class FormulaCalculate {

    private static final Map<String, Integer> optToPri = Map.of(
            "+", 1,
            "-", 1,
            "*", 2,
            "/", 2,
            "^", 3
    );

    public static void main(String[] args) {
        String formula = "(101 * 88 + (155 / 23 + 12^2)) / 999";
        List<String> formulaList = parseFormula(formula);
        BigDecimal result = calcFormulaWithoutBrackets(formulaList);
        System.out.println(result.toPlainString()); // = 9.04
        /*
         * 额外示例：
         * (11 - 2 + (15 / 3 + 12 / 2)) * 99 = -198.00
         * (101 * 88 + (15 / 3 + 12 / 2)) / 1888 = 4.71
         * 11 * 2 - (15 / 3 + 12 * 2) * 99 = -2849.00
         */
    }

    private static List<String> parseFormula(String formula) {
        List<String> formulaList = new ArrayList<>(formula.length());
        LinkedList<String> tmpQueue = new LinkedList<>();
        char[] formulaCharArray = formula.toCharArray();
        for (char c : formulaCharArray) {
            // 忽略空格
            if (c == 32) {
                continue;
            }
            if (Character.isDigit(c)) {
                tmpQueue.add(String.valueOf(c));
            } else {
                String e = "";
                String chr;
                while ((chr = tmpQueue.pollFirst()) != null) {
                    e += chr;
                }
                formulaList.add(e);
                formulaList.add(String.valueOf(c));
            }
        }
        if (!tmpQueue.isEmpty()) {
            String e = "";
            String chr;
            while ((chr = tmpQueue.pollFirst()) != null) {
                e += chr;
            }
            formulaList.add(e);
        }
        return formulaList;
    }

    private static BigDecimal calcFormulaWithoutBrackets(List<String> formulaList) {
        Stack<BigDecimal> stack1 = new Stack<>();
        Stack<String> stack2 = new Stack<>();
        for (String e : formulaList) {
            try {
                // 数字直接入栈
                stack1.push(new BigDecimal(e)); // 或者使用 isNumeric() 方法来判断数字
            } catch (NumberFormatException ex) {
                // 非数字，则判断符号
                if (optToPri.containsKey(e)) {
                    if (stack2.isEmpty()) {
                        stack2.push(e);
                        continue;
                    }
                    String opt = stack2.peek();
                    if (Objects.equals(opt, "(")) {
                        stack2.push(e);
                        continue;
                    }
                    // 如果前一个的操作符优先级比当前操作符优先级高，则计算stack1中的最后两个数，并将结果压入stack1，并将当前操作符压入stack2
                    if (optToPri.get(opt) > optToPri.get(e)) {
                        stack1.push(doActualCalc(stack1, opt));
                        stack2.pop();
                    }
                    stack2.push(e);
                    continue;
                }
                if (Objects.equals(e, "(")) {
                    stack2.push(e);
                    continue;
                }
                if (Objects.equals(e, ")")) {
                    while (!stack2.empty()) {
                        String opt = stack2.pop();
                        if (opt.equals("(")) {
                            // 左括号出栈，结束
                            break;
                        }
                        stack1.push(doActualCalc(stack1, opt));
                    }
                }
            }
        }
        // 结束之后，再处理栈中还剩下的元素
        while (!stack2.isEmpty()) {
            String opt = stack2.pop();
            stack1.push(doActualCalc(stack1, opt));
        }
        return stack1.pop();
    }

    private static boolean isNumeric(String str) {
        if (str == null) {
            return false;
        }
        int sz = str.length();
        for (int i = 0; i < sz; i++) {
            if (!Character.isDigit(str.charAt(i))) {
                return false;
            }
        }
        return true;
    }

    private static BigDecimal doActualCalc(Stack<BigDecimal> stack, String opt) {
        BigDecimal num1 = stack.pop();
        BigDecimal num2 = stack.pop();
        return switch (opt) {
            case "+" -> num2.add(num1);
            case "-" -> num2.subtract(num1);
            case "*" -> num2.multiply(num1);
            case "/" -> num2.divide(num1, 2, RoundingMode.DOWN);
            case "^" -> num2.pow(num1.intValue());
            default -> throw new RuntimeException("Undeclared opt: " + opt);
        };
    }

}
```

