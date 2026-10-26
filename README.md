from os import system
import itertools

#Es un programa que reciba un valor de resistencia y obtenga el valor comercial más próximo usando 1, 2 o 3 resistencias en todas las combinaciones posibles serie paralelo en que se puedan conectar (con tolerancia de1% o 10%) 

class AnalizadorResistencias:
    """Clase encargada de la lógica matemática y búsqueda de valores comerciales."""
    
    E12 = [1.0, 1.2, 1.5, 1.8, 2.2, 2.7, 3.3, 3.9, 4.7, 5.6, 6.8, 8.2]
    E24 = [
        1.0, 1.1, 1.2, 1.3, 1.5, 1.6, 1.8, 2.0, 2.2, 2.4, 
        2.7, 3.0, 3.3, 3.6, 3.9, 4.3, 4.7, 5.1, 5.6, 6.2, 
        6.8, 7.5, 8.2, 9.1
    ]

    def __init__(self, tolerancia=10):
        self.tolerancia = tolerancia

    def obtener_valores_comerciales(self):
        """Genera el arreglo de valores comerciales en todas las décadas."""
        base = self.E12 if self.tolerancia == 10 else self.E24
        valores = []
        for decade in range(-2, 8):  # Desde 0.01 ohms hasta 10 Megaohms
            for b in base:
                valores.append(round(b * (10 ** decade), 6))
        return valores

    def calcular_mejor_combinacion(self, target):
        """Busca la mejor combinación de 1, 2 o 3 resistencias (serie/paralelo)."""
        if target <= 0:
            raise ValueError("El valor objetivo debe ser mayor a 0.")

        all_vals = self.obtener_valores_comerciales()

        filtered_vals = [v for v in all_vals if (target / 50.0) <= v <= (target * 50.0)]
        if not filtered_vals:
            filtered_vals = all_vals

        mejor_diff = float("inf")
        mejor_resultado = None

        def evaluar(val, tipo, componentes):
            nonlocal mejor_diff, mejor_resultado
            diff = abs(val - target)
            if diff < mejor_diff:
                mejor_diff = diff
                pct_error = (diff / target) * 100
                mejor_resultado = {
                    "equivalente": val,
                    "error_pct": pct_error,
                    "tipo": tipo,
                    "componentes": componentes,
                }

        # 1. Una sola resistencia
        for r1 in filtered_vals:
            evaluar(r1, "Única", [r1])

        # 2. Dos resistencias
        for r1, r2 in itertools.combinations_with_replacement(filtered_vals, 2):
            evaluar(r1 + r2, "Serie (2)", [r1, r2])
            if (r1 + r2) > 0:
                evaluar((r1 * r2) / (r1 + r2), "Paralelo (2)", [r1, r2])

        # 3. Tres resistencias
        for r1, r2, r3 in itertools.combinations_with_replacement(filtered_vals, 3):
            evaluar(r1 + r2 + r3, "Serie (3)", [r1, r2, r3])

            denom = (1 / r1) + (1 / r2) + (1 / r3)
            if denom > 0:
                evaluar(1 / denom, "Paralelo (3)", [r1, r2, r3])

            # Mixtas: (R1 || R2) + R3 y permutaciones
            p1 = (r1 * r2) / (r1 + r2) if (r1 + r2) > 0 else 0
            evaluar(p1 + r3, "Mixta ((R1 || R2) + R3)", [r1, r2, r3])

            p2 = (r1 * r3) / (r1 + r3) if (r1 + r3) > 0 else 0
            evaluar(p2 + r2, "Mixta ((R1 || R3) + R2)", [r1, r3, r2])

            p3 = (r2 * r3) / (r2 + r3) if (r2 + r3) > 0 else 0
            evaluar(p3 + r1, "Mixta ((R2 || R3) + R1)", [r2, r3, r1])

            # Mixtas: (R1 + R2) || R3 y permutaciones
            s1 = r1 + r2
            if (s1 + r3) > 0:
                evaluar((s1 * r3) / (s1 + r3), "Mixta ((R1 + R2) || R3)", [r1, r2, r3])

            s2 = r1 + r3
            if (s2 + r2) > 0:
                evaluar((s2 * r2) / (s2 + r2), "Mixta ((R1 + R3) || R2)", [r1, r3, r2])

            s3 = r2 + r3
            if (s3 + r1) > 0:
                evaluar((s3 * r1) / (s3 + r1), "Mixta ((R2 + R3) || R1)", [r2, r3, r1])

        return mejor_resultado


class MenuInteractivo:
    """Clase encargada de la interfaz de usuario y la navegación por consola."""
    
    @staticmethod
    def mostrar_portada():
        system("cls")
        system("color 0A")
        print("--------------------------------------")
        print("|        OSCILACIONES Y ONDAS        |")
        print("|            GRUPO                   |")
        print("|              Equipo                |")
        print("|                 J                  |")
        print("--------------------------------------")
        system("pause")

    @staticmethod
    def mostrar_codigo_colores():
        system("cls")
        print("==================================================")
        print("         TABLA DE CÓDIGO DE COLORES               ")
        print("==================================================")
        print(f"{'Color':<12} | {'Banda 1 & 2':<12} | {'Multiplicador':<15} | {'Tolerancia'}")
        print("-" * 62)
        print(f"{'Negro':<12} | {0:<12} | {'1 Ω (10⁰)':<15} | -")
        print(f"{'Café / Marrón':<12} | {1:<12} | {'10 Ω (10¹)':<15} | ±1% (F)")
        print(f"{'Rojo':<12} | {2:<12} | {'100 Ω (10²)':<15} | ±2% (G)")
        print(f"{'Naranja':<12} | {3:<12} | {'1 kΩ (10³)':<15} | -")
        print(f"{'Amarillo':<12} | {4:<12} | {'10 kΩ (10⁴)':<15} | -")
        print(f"{'Verde':<12} | {5:<12} | {'100 kΩ (10⁵)':<15} | ±0.5% (D)")
        print(f"{'Azul':<12} | {6:<12} | {'1 MΩ (10⁶)':<15} | ±0.25% (C)")
        print(f"{'Violeta':<12} | {7:<12} | {'10 MΩ (10⁷)':<15} | ±0.1% (B)")
        print(f"{'Gris':<12} | {8:<12} | {'100 MΩ (10⁸)':<15} | ±0.05% (A)")
        print(f"{'Blanco':<12} | {9:<12} | {'1 GΩ (10⁹)':<15} | -")
        print(f"{'Dorado':<12} | {'-':<12} | {'0.1 (10⁻¹)':<15} | ±5% (J)")
        print(f"{'Plateado':<12} | {'-':<12} | {'0.01 (10⁻²)':<15} | ±10% (K)")
        print("==================================================")
        input("\nPresione Enter para regresar al menú principal...")

    def ejecutar(self):
        self.mostrar_portada()
        
        while True:
            system("cls")
            print("========================================")
            print("          MENÚ PRINCIPAL - R/C          ")
            print("========================================")
            print("1. Calcular combinación de resistencias")
            print("2. Visualizar código de colores")
            print("3. Salir")
            print("========================================")
            
            opcion = input("Seleccione una opción (1-3): ").strip()
            
            if opcion == "1":
                system("cls")
                print("--- CÁLCULO DE COMBINACIÓN DE RESISTENCIAS ---")
                try:
                    valor_deseado = float(input("\nIngrese el valor de resistencia deseado (ohms): "))
                    tolerancia_deseada = int(input("Ingrese la tolerancia deseada (1 o 10 %): "))

                    if tolerancia_deseada not in [1, 10]:
                        print("\n[!] Tolerancia no válida. Usando por defecto 10%.")
                        tolerancia_deseada = 10

                    analizador = AnalizadorResistencias(tolerancia_deseada)
                    resultado = analizador.calcular_mejor_combinacion(valor_deseado)

                    print("\n--- Resultado Óptimo ---")
                    print(f"Valor objetivo: {valor_deseado} Ω")
                    print(f"Tolerancia de serie comercial: {tolerancia_deseada}%")
                    print(f"Valor equivalente obtenido: {resultado['equivalente']:.4f} Ω")
                    print(f"Error porcentual: {resultado['error_pct']:.4f}%")
                    print(f"Topología / Conexión: {resultado['tipo']}")
                    print(f"Valores de las resistencias usadas: {resultado['componentes']}")
                
                except ValueError as e:
                    print(f"\n[Error] Entrada inválida: {e}")
                
                input("\nPresione Enter para continuar...")

            elif opcion == "2":
                self.mostrar_codigo_colores()

            elif opcion == "3":
                system("cls")
                print("\nSaliendo del programa. ¡Hasta luego!")
                break
            else:
                input("\n[!] Opción no válida. Presione Enter para intentar de nuevo...")


if __name__ == "__main__":
    app = MenuInteractivo()
    app.ejecutar()
