---
layout:     post
title:      "用高斯消元法求解线性方程组"
date:       2016-03-20 12:00:00
author:     "Thh"
tags:
    - 算法
---

线性方程组问题可以利用矩阵变换求解。利用高斯消元法，将矩阵转换成一个行阶梯矩阵，最后得到一个简化行阶梯矩阵，就是方程的解。参考资料（[高斯消元法](https://zh.wikipedia.org/wiki/%E9%AB%98%E6%96%AF%E6%B6%88%E5%8E%BB%E6%B3%95)）

## Java代码

	public class FunctionResolver {

    	public static class LinearEquationGroup {
    		/*
    		  代表线性方程组的矩阵。方程组已经经过归一化处理，带未知变量的部分全部位于“=”左边，常数合并后位于“=”右边。
    		  比如： 
    		  	2a + b - c = 8
            	-3a - b + 2c = -11
            	-2a + b + 2c = -3
            	
              对应的矩阵为：
               2   1  -1    8
              -3  -1   2  -11
              -2   1   2   -3
    		*/
	        private BigDecimal[][] matrix;
	        
	        //未知变量的名称，排列顺序和矩阵一致，比如上面的例子中，对应的变量名列表就是a, b, c
	        private List<String> variantes;

	        public LinearEquationGroup() {
	        }

	        public LinearEquationGroup(BigDecimal[][] matrix) {
	            this.matrix = matrix;
	            check();
	        }

			private void check() {
            	for (int i = 0; i < matrix.length; i++) {
                if (matrix.length != (matrix[i].length - 1)) {
                    throw new IllegalArgumentException("输入矩阵有误! 必须为n*(n+1)矩阵");
                	}
            	}
        	}
        	
	        public Map<String, BigDecimal> solve() {
	        	check();
	            sortRows();
	            eliminateVarianteDownwards();
	            normalize();
	            eliminateVarianteUpwards();
	            Map<String, BigDecimal> ret = new HashMap<>();
	            int i = 0;
	            int lastCol = variantes.size();
	            for (String var : variantes) {
	                ret.put(var, matrix[i++][lastCol]);
	            }
	            return ret;
	        }

	        // 重排序行, 以便做高斯消元. 保证第i行的第i列元素不为0
	        void sortRows() {
	            int row = 0;
	            int below = 0;
	            int col = 0;
	            for (; row < matrix.length - 1; ++row) {
	                col = row;
	                if (matrix[row][col].compareTo(BigDecimal.ZERO) == 0) {
	                    for (below = row + 1; below < matrix.length - 1; below++) {
	                        if (matrix[below][col].compareTo(BigDecimal.ZERO) != 0) {
	                            BigDecimal[] temp = matrix[row];
	                            matrix[row] = matrix[below];
	                            matrix[below] = temp;
	                            break;
	                        }
	                    }
	                    if (below >= matrix.length) {
                        	throw new IllegalArgumentException("方程组无解或者无唯一解!");
                    	}
	                }
	            }
	        }

			//从上往下消元。消元结果使得第i行以下的第i列都为0
	        void eliminateVarianteDownwards() {
	            int baseRow = 0;
	            int targetRow;
	            final BigDecimal minusOne = new BigDecimal(-1);
	            int colCnt = matrix[0].length;
	            int rowCnt = matrix.length;
	            for (; baseRow < matrix.length - 1; ++baseRow) {
	                int col = baseRow;
	                for (targetRow = baseRow + 1; targetRow < matrix.length; ++targetRow) {
	                    if (matrix[targetRow][col].compareTo(BigDecimal.ZERO) != 0) {
	                        BigDecimal fraction = matrix[targetRow][col].divide(matrix[baseRow][col]).multiply(minusOne);
	                        for (int j = 0; j < colCnt; j++) {
	                            matrix[targetRow][j] = matrix[targetRow][j].add(fraction.multiply(matrix[baseRow][j]));
	                        }
	                    }
	                }
	            }
	        }

			//归一化。使得第i行第i列的元素都为1，这样最后得到的简化行阶梯矩阵的最后一列即为解
	        void normalize() {
	            for (int i = 0; i < matrix.length; i++) {
	                if (matrix[i][i].compareTo(BigDecimal.ZERO) == 0) {
	                    throw new IllegalArgumentException("矩阵对角线的元素不应该为0, 输入有误或者程序有bug!");
	                }
	                if (matrix[i][i].compareTo(BigDecimal.ONE) == 0)
	                    continue;
	                for (int j = matrix[i].length - 1; j >= i; j--) {
	                    matrix[i][j] = matrix[i][j].divide(matrix[i][i]);
	                }
	            }
	        }

			//从下往上消元。使得第i行以上第i列的元素都为0
	        void eliminateVarianteUpwards() {
	            int baseRow = matrix.length - 1;
	            int colCnt = matrix[0].length;
	            final BigDecimal minusOne = new BigDecimal(-1);
	            for (; baseRow > 0; baseRow--) {
	                int col = baseRow;
	                for (int targetRow = baseRow - 1; targetRow >= 0; targetRow--) {
	                    if (matrix[targetRow][col].compareTo(BigDecimal.ZERO) != 0) {
	                        BigDecimal fraction = minusOne.multiply(matrix[targetRow][col].divide(matrix[baseRow][col]));
	                        for (int j = 0; j < colCnt; j++) {
	                            matrix[targetRow][j] = matrix[targetRow][j].add(fraction.multiply(matrix[baseRow][j]));
	                        }
	                    }
	                }
	            }
	        }

	        public BigDecimal[][] getMatrix() {
	            return matrix;
	        }

	        public List<String> getVariantes() {
	            return variantes;
	        }

	        public void setVariantes(List<String> variantes) {
	            this.variantes = variantes;
	        }

	        @Override
	        public boolean equals(Object o) {
	            if (this == o) return true;
	            if (o == null || getClass() != o.getClass()) return false;

	            LinearEquationGroup that = (LinearEquationGroup) o;

	            return Arrays.deepEquals(matrix, that.matrix);

	        }

	        @Override
	        public int hashCode() {
	            return Arrays.deepHashCode(matrix);
	        }
	    }

		//解析Map形式输入的方程组。输入类型Map<String, String>，每个entry代表一个方程，entry的key是方程“=”左边部分，entry的value是“=”右边部分。未知变量用字母串表示。 
	    public static class LinearEquationGroupBuilder {
	        private Map<String, List<String>> keyTokens = new HashMap<>();
	        private Map<String, List<String>> valueTokens = new HashMap<>();

	        public LinearEquationGroup createFrom(Map<String, String> funcMap) {
	            final LinearEquationGroup linearEquationGroup = new LinearEquationGroup();
	            linearEquationGroup.matrix = new BigDecimal[funcMap.size()][];
	            funcMap.forEach((k, v) -> {
	                keyTokens.put(k, tokenize(k));
	                valueTokens.put(v, tokenize(v));
	            });
	            Map<String, Integer> vars = findVariantes(funcMap);
	            final ArrayList<String> variants = new ArrayList<>(vars.size());
	            vars.forEach((k, v) -> variants.add(v, k));
	            linearEquationGroup.setVariantes(variants);
	            BigDecimal[][] matrix = linearEquationGroup.matrix;
	            for (int i = 0; i < matrix.length; i++) {
	                matrix[i] = new BigDecimal[vars.size() + 1];
	                for (int j = 0; j < matrix[i].length; j++) {
	                    matrix[i][j] = new BigDecimal(0);
	                }
	            }
	            buildMatrix(matrix, funcMap, vars);
	            return linearEquationGroup;
	        }

	        private Map<String, Integer> findVariantes(Map<String, String> fucMap) {
	            final Map<String, Integer> map = new HashMap<>();
	            final int[] keyIdx = new int[1];
	            keyIdx[0] = 0;
	            fucMap.forEach((k,v) -> {
	                parseVariantes(map, keyIdx, keyTokens.get(k));
	                parseVariantes(map, keyIdx, valueTokens.get(v));
	            });
	            return map;
	        }

	        private void parseVariantes(Map<String, Integer> map, int[] keyIdx, List<String> tokens) {
	            for (String part : tokens) {
	                final Object[] objects = splitFactorAndVar(part);
	                if (objects[1] != null) {
	                    final String var = (String)objects[1];
	                    if (!map.containsKey(var)) {
	                        map.put(var, keyIdx[0]++);
	                    }
	                }
	            }
	        }

	        private LinkedList<String> tokenize(String k) {
	            final LinkedList<String> parts = new LinkedList<>();
	            int end = k.length();
	            for (int i = k.length() - 1; i >= 0; i--) {
	                if (k.charAt(i) == '-') {
	                    final String trimed = k.substring(i, end).trim();
	                    if (trimed.length() > 0)
	                        parts.addFirst(trimed);
	                    end = i;
	                } else if (k.charAt(i) == '+') {
	                    final String trimed = k.substring(i + 1, end).trim();
	                    if (trimed.length() > 0)
	                        parts.addFirst(trimed);
	                    end = i;
	                } else {
	                    //
	                }
	            }
	            if (end > 0) {
	                final String trimed = k.substring(0, end).trim();
	                if (trimed.length() > 0)
	                    parts.addFirst(trimed);
	            }
	            return parts;
	        }

	        /**
	         * 解析数字和变量
	         * @param part 数字和变量, 或者纯数字, 或者纯变量, 比如2a, 1.5b, 7, c, -3d, -e
	         * @return 返回一个数组, 0号元素是解析出来的系数, 1号元素是变量名. 系数或变量名可能不存在,这个时候对应位置的元素为null
	         */
	        Object[] splitFactorAndVar(String part) {
	            Object[] ret = new Object[]{null, null};
	            part = part.trim();
	            boolean isNegative = false;
	            if (part.charAt(0) == '-') {
	                isNegative = true;
	                part = part.substring(1);
	            }
	            int idx = part.length() - 1;
	            while (idx >= 0 && !Character.isDigit(part.charAt(idx))) --idx;
	            if (idx >= 0) {
	                double factor = Double.parseDouble(part.substring(0, idx + 1));
	                ret[0] = factor;
	                String var = null;
	                if (idx + 1 < part.length()) {
	                    var = part.substring(idx + 1);
	                    ret[1] = var.trim();
	                }
	            } else {
	                ret[1] = part.trim();
	                ret[0] = 1.0;
	            }
	            if (isNegative) {
	                ret[0] = -1 * (double)ret[0];
	            }
	            return ret;
	        }

	        private void buildMatrix(BigDecimal[][] matric, Map<String, String> funcMap, Map<String, Integer> varIds) {
	            final int[] i = new int[1];
	            i[0] = 0;
	            funcMap.forEach((k, v) -> {
	                //matric[i[0]] = new BigDecimal[varIds.size() + 1];
	                processOneFunction(matric[i[0]++], k, v, varIds);
	            });
	        }

	        void processOneFunction(BigDecimal[] matricRow, String key, String value, Map<String, Integer> varIds) {
	            processOneside(matricRow, key, varIds, true);
	            processOneside(matricRow, value, varIds, false);
	        }

	        private void processOneside(BigDecimal[] matricRow, String key, Map<String, Integer> varIds, boolean isKey) {
	            int commonFactor;
	            List<String> parts;
	            if (isKey) {
	                commonFactor = 1;
	                parts = keyTokens.get(key);
	            } else {
	                commonFactor = -1;
	                parts = valueTokens.get(key);
	            }
	            for (String part : parts) {
	                final Object[] objects = splitFactorAndVar(part);
	                // 含有变量
	                if (objects[1] != null) {
	                    if (objects[0] != null) {
	                        final Double factor = (Double) objects[0];
	                        final Integer idx = varIds.get((String) objects[1]);
	                        if (idx == null) {
	                            throw new IllegalArgumentException("can't found " + objects[1] + " in varIds");
	                        }
	                        if (matricRow[idx] == null) {
	                            matricRow[idx] = new BigDecimal(factor * commonFactor);
	                        } else {
	                            matricRow[idx] = matricRow[idx].add(new BigDecimal(factor * commonFactor));
	                        }
	                    } else {
	                        final Integer idx = varIds.get((String) objects[1]);
	                        if (idx == null) {
	                            throw new IllegalArgumentException("can't found " + objects[1] + " in varIds");
	                        }
	                        if (matricRow[idx] == null) {
	                            matricRow[idx] = new BigDecimal(1 * commonFactor);
	                        } else {
	                            matricRow[idx] = matricRow[idx].add(new BigDecimal(1 * commonFactor));
	                        }
	                    }
	                } else {
	                    //只是数字
	                    if (objects[0] != null) {
	                        matricRow[matricRow.length - 1] = matricRow[matricRow.length - 1].add(new BigDecimal(-1 * commonFactor * ((Double)objects[0])));
	                    } else {
	                        throw new IllegalArgumentException("系数和数字不能同时为null: " + key);
	                    }
	                }
	            }
	        }
	    }

	    public static void main(String[] args) {
	        final FunctionResolver.LinearEquationGroupBuilder linearEquationGroupBuilder = new FunctionResolver.LinearEquationGroupBuilder();
	        Map<String, String> fMap = new HashMap<>();
	        fMap.put("2a+b-c", "8");
	        fMap.put("-3a-b+2c", "-11");
	        fMap.put("-2a+b+2c", "-3");
	        FunctionResolver.LinearEquationGroup equationGroup = linearEquationGroupBuilder.createFrom(fMap);
	        final Map<String, BigDecimal> solve = equationGroup.solve();
	        solve.forEach((k, v) -> System.out.println(k + "=" + v));
	    }
	}
	
## 复杂度分析
该算法的时间复杂度为O(n^3)，空间复杂度为O(n^2)。对于维度不高的线性方程还是可以接受。