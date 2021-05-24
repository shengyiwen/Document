## Type类介绍

Type is the common superinterface for all types in the Java programming language. These include raw types, parameterized types, array types, type variables and primitive types.

Type类是java中所有类型的父接口，包括如下类型

- Raw Types（原始类型）：包括类、枚举、注解、接口、数组[但不包括泛型数组]
- Parameterized Types（参数类型）：如Set<String>, Map<String, String>, Class<?> 等
- Array Types（泛型数组和参数类型数组）：如T[]，List<String>[]
- Type Variables（类型变量）：如T，K，V
- Primitive Types（基本类型）：如int, double等


    import java.lang.annotation.*;
    import java.lang.reflect.*;
    import java.util.List;
    import java.util.Map;

    /**
    * @author asheng
    * @since 2019/6/24
    */
    public class TypeTest<T> {

        enum EnumType {
        }

        interface InterfaceType {
        }

        class ClassType {
        }

        @Documented
        @Retention(RetentionPolicy.RUNTIME)
        @Target({ElementType.FIELD})
        @interface AnnotationType {
        }


        //TypeVariable
        private T t;

        //RawType --> class
        private EnumType enumType;

        //RawType --> class
        private ClassType classType;

        //RawType ---> class
        private AnnotationType annotationType;

        //ParameterizedType
        private Class<?> clazz;

        //GenericArrayType
        private T[] tArray;

        //RawType --> int
        private int a;

        //RawType --> class
         private Integer b;

        //RawType --> class
         private String c;

        //RawType --> class
        private int[] aArray;

        //RawType --> class
        private Integer[] bArray;
    
        //RawType --> clazz
        private String[] cArray;

        //ParameterizedType
        private List<String> list;

        //ParameterizedType
        private List<T> tList;

        //ParameterizedType
        private Map<String, String> map;

        //ParameterizedType
        private Map<?, ?> clazzMap;

        //ParameterizedType
        private Map<T, T> tMap;

        //GenericArrayType
        private List<T>[] tListArray;

        //ParameterizedType
        private TypeTest<T> tType;

        //GenericArrayType
        private TypeTest<T>[] tTypeArray;

        public static void main(String[] args) {
            Field[] fields = TypeTest.class.getDeclaredFields();

            for (Field field : fields) {
                System.out.println("变量名: " + field.getName());
                System.out.println("变量类型: " + field.getType());
                Type type = field.getGenericType();
                System.out.println("变量的声明类型: " + type.getTypeName());

                if (type instanceof Class) {
                    System.out.println("变量类型是Class");
                 } else if (type instanceof ParameterizedType) {
                    System.out.println("变量类型是ParameterizedType");
                } else if (type instanceof TypeVariable) {
                    System.out.println("变量类型TypeVariable");
                } else if (type instanceof GenericArrayType) {
                    System.out.println("变量类型GenericArrayType");
                } else if (type instanceof WildcardType) {
                    System.out.println("变量类型WildCardType");
                }
             }
        }

    }
