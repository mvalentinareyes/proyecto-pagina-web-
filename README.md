# proyecto-pagina-web-
En este proyecto desarrollé BudAppest, una app en Python y Streamlit conectada a Oracle Cloud para la gestión hotelera. Implementé módulos para consultar habitaciones, buscar reservas con SQL, registrar huéspedes y calcular costos. También añadí control de pagos, cálculo de saldos pendientes con funciones de SQL y actualización automática.
# ----------------- IMPORTAMOS LIBRERÍAS -----------------
import streamlit as st
import pandas as pd
from datetime import date
import oracledb

# ----------------- CONFIGURACIÓN -----------------
# Aquí configuro la página principal de la aplicación
st.set_page_config(page_title="BudAppest", layout="wide")

st.title("Hotel Budapest")
st.write("Esta aplicación gestiona el sistema de reservas del Hotel Budapest")

# ----------------- CONEXIÓN A ORACLE -----------------
# Esta función me permite conectarme a la base de datos en Oracle Cloud
def conectar():
    return oracledb.connect(
        user="G04_E02",
        password="Bi!2026-04-02",
        dsn="adbg04_low"
    )

# Esta función sirve para ejecutar consultas SQL
# y traer la información en un DataFrame de pandas
def consultar(sql, params=None):
    conn = conectar()
    df = pd.read_sql(sql, conn, params=params)
    conn.close()
    return df

# ----------------- MENÚ -----------------
# Menú lateral para navegar entre las opciones de la app
st.sidebar.header("Menú")

pagina = st.sidebar.radio("Seleccione una opción", [
    "🛌 Habitaciones disponibles",
    "🔎 Consultar reservas",
    "📅 Registrar nueva reserva",
    "💵 Registrar pago"
])

# Encabezado visual
col_1, col_2 = st.columns(2)

with col_1:
    st.write("Sistema de gestión de reservas BudAppest")

with col_2:
    st.write(f"Sección activa: *{pagina}*")

st.divider()

# ============================================================
# 🛌 CONSULTA 1: HABITACIONES DISPONIBLES
# ============================================================
# Aquí muestro solamente las habitaciones que están disponibles
if pagina == "🛌 Habitaciones disponibles":

    st.subheader("🛌 Habitaciones disponibles")

    df_habitaciones = consultar("""
        SELECT
            num_habitacion,
            tipo,
            precio_noche
        FROM ht_habitaciones
        WHERE estado = 'DISPONIBLE'
    """)

    st.write(f"Total de habitaciones disponibles: *{len(df_habitaciones)}*")

    st.dataframe(
        df_habitaciones.rename(columns={
            "NUM_HABITACION": "Número",
            "TIPO": "Tipo",
            "PRECIO_NOCHE": "Precio por noche"
        }),
        use_container_width=True,
        hide_index=True
    )

# ============================================================
# 🔎 CONSULTA 2: CONSULTAR RESERVAS
# ============================================================
# Aquí uno varias tablas para mostrar la información completa
elif pagina == "🔎 Consultar reservas":

    st.subheader("🔎 Reservas registradas")

    estado = st.selectbox(
        "Filtrar por estado",
        ["ACTIVA", "CANCELADA", "PAGÓ"]
    )

    df_reservas = consultar("""
        SELECT
            r.reserva_id,
            h.nombre,
            ha.num_habitacion,
            r.fecha_ingreso,
            r.fecha_salida,
            r.monto_total,
            r.estado
        FROM ht_reservas r
        JOIN ht_huespedes h
            ON r.huesped_id = h.huesped_id
        JOIN ht_habitaciones ha
            ON r.habitacion_id = ha.habitacion_id
        WHERE r.estado = :estado
    """, params={"estado": estado})

    st.write(f"Reservas encontradas: **{len(df_reservas)}**")

    st.dataframe(
        df_reservas.rename(columns={
            "RESERVA_ID": "ID",
            "NOMBRE": "Huésped",
            "NUM_HABITACION": "Habitación",
            "FECHA_INGRESO": "Entrada",
            "FECHA_SALIDA": "Salida",
            "MONTO_TOTAL": "Total",
            "ESTADO": "Estado"
        }),
        use_container_width=True,
        hide_index=True
    )

# ============================================================
# 📅 CONSULTA 3: REGISTRAR NUEVA RESERVA
# ============================================================
# En esta parte el usuario puede registrar una nueva reserva
elif pagina == "📅 Registrar nueva reserva":

    st.subheader("📅 Nueva reserva")

    # Traigo las habitaciones disponibles
    df_habitaciones = consultar("""
        SELECT
            habitacion_id,
            num_habitacion,
            tipo,
            precio_noche
        FROM ht_habitaciones
        WHERE estado = 'DISPONIBLE'
    """)

    # Creo el texto que se mostrará en el selectbox
    df_habitaciones["opcion"] = (
        df_habitaciones["NUM_HABITACION"] + " - " +
        df_habitaciones["TIPO"] + " ($" +
        df_habitaciones["PRECIO_NOCHE"].astype(str) + ")"
    )

    col_3, col_4 = st.columns(2)

    with col_3:
        num_doc = st.text_input("Número de documento")

        habitacion = st.selectbox(
            "Habitación",
            df_habitaciones["opcion"].tolist()
        )

        fecha_entrada = st.date_input(
            "Fecha de entrada",
            value=date.today()
        )

    with col_4:
        nombre = st.text_input("Nombre del huésped")

        canal = st.selectbox(
            "Canal",
            ["TRANSFERENCIA", "PSE", "TARJETA"]
        )

        fecha_salida = st.date_input(
            "Fecha de salida",
            value=date.today()
        )

    # Obtengo los datos de la habitación seleccionada
    hab_seleccionada = df_habitaciones[
        df_habitaciones["opcion"] == habitacion
    ]

    if not hab_seleccionada.empty:

        precio = int(
            hab_seleccionada["PRECIO_NOCHE"].values[0]
        )

        habitacion_id = int(
            hab_seleccionada["HABITACION_ID"].values[0]
        )

        noches = max(
            (fecha_salida - fecha_entrada).days,
            0
        )

        total = noches * precio

        st.info(
            f"{noches} noches x ${precio:,} = ${total:,}"
        )

        # Botón para registrar la reserva
        if st.button("Registrar reserva", type="primary"):

            if not num_doc or not nombre:

                st.warning(
                    "Completa el nombre y el número de documento"
                )

            elif fecha_salida <= fecha_entrada:

                st.warning(
                    "La fecha de salida debe ser posterior a la entrada"
                )

            else:

                try:

                    conn = conectar()
                    cursor = conn.cursor()

                    # Primero reviso si el huésped ya existe
                    cursor.execute("""
                        SELECT huesped_id
                        FROM ht_huespedes
                        WHERE num_doc = :1
                    """, [num_doc])

                    fila = cursor.fetchone()

                    # Si el huésped ya existe uso el mismo ID
                    if fila:

                        huesped_id = fila[0]

                    else:

                        # ACÁ SE REGISTRA CON ORACLE
                        # Si el huésped no existe lo inserto
                        cursor.execute("""
                            INSERT INTO ht_huespedes
                            (tipo_doc, num_doc, nombre, estado)
                            VALUES ('CC', :1, :2, 'ACTIVO')
                        """, [num_doc, nombre])

                        cursor.execute("""
                            SELECT huesped_id
                            FROM ht_huespedes
                            WHERE num_doc = :1
                        """, [num_doc])

                        huesped_id = cursor.fetchone()[0]

                    # ACÁ SE REGISTRA CON ORACLE
                    # Aquí inserto la nueva reserva
                    cursor.execute("""
                        INSERT INTO ht_reservas
                        (
                            huesped_id,
                            habitacion_id,
                            fecha_ingreso,
                            fecha_salida,
                            monto_total,
                            estado
                        )
                        VALUES
                        (
                            :1,
                            :2,
                            :3,
                            :4,
                            :5,
                            'ACTIVA'
                        )
                    """, [
                        huesped_id,
                        habitacion_id,
                        fecha_entrada,
                        fecha_salida,
                        total
                    ])

                    conn.commit()
                    conn.close()

                    st.success(
                        f"Reserva registrada para {nombre} "
                        f"por un total de ${total:,}"
                    )

                except Exception as e:

                    st.error(f"Error: {e}")

# ============================================================
# CONSULTA 4: REGISTRAR PAGO
# ============================================================
# Aquí calculo el saldo pendiente y permito registrar pagos
elif pagina == "💵 Registrar pago":

    st.subheader("💵 Registrar pago")

    reservas_con_saldo = consultar("""
        SELECT
            r.reserva_id,
            h.nombre,
            ha.num_habitacion,
            r.monto_total,
            NVL(SUM(p.monto_pago), 0) AS total_pagado,
            r.monto_total -
            NVL(SUM(p.monto_pago), 0) AS saldo_pendiente
        FROM ht_reservas r

        JOIN ht_huespedes h
            ON r.huesped_id = h.huesped_id

        JOIN ht_habitaciones ha
            ON r.habitacion_id = ha.habitacion_id

        LEFT JOIN ht_pagos p
            ON r.reserva_id = p.reserva_id

        WHERE r.estado = 'ACTIVA'

        GROUP BY
            r.reserva_id,
            h.nombre,
            ha.num_habitacion,
            r.monto_total

        HAVING
            r.monto_total -
            NVL(SUM(p.monto_pago), 0) > 0
    """)

    if reservas_con_saldo.empty:

        st.info("No hay reservas con saldo pendiente")

    else:

        etiquetas = (
            reservas_con_saldo["RESERVA_ID"].astype(str)
            + " — "
            + reservas_con_saldo["NOMBRE"]
        )

        reserva_sel = st.selectbox(
            "Reserva",
            etiquetas.tolist()
        )

        fila = reservas_con_saldo[
            etiquetas == reserva_sel
        ].iloc[0]

        saldo = float(fila["SALDO_PENDIENTE"])

        st.write(f"Saldo pendiente: **${saldo:,.0f}**")

        col_1, col_2 = st.columns(2)

        with col_1:

            monto = st.number_input(
                "Monto del pago ($)",
                min_value=1000.0,
                max_value=saldo,
                value=saldo,
                step=10000.0
            )

        with col_2:

            medio = st.selectbox(
                "Medio de pago",
                [
                    "EFECTIVO",
                    "TARJETA_CREDITO",
                    "TRANSFERENCIA",
                    "PSE"
                ]
            )

        referencia = st.text_input("Referencia del pago")

        if st.button("Registrar pago", type="primary"):

            if not referencia:

                st.warning(
                    "Ingresa una referencia para el pago"
                )

            else:

                try:

                    conn = conectar()
                    cursor = conn.cursor()

                    # ACÁ SE REGISTRA CON ORACLE
                    # Aquí inserto el pago en la tabla
                    cursor.execute("""
                        INSERT INTO ht_pagos
                        (
                            reserva_id,
                            fecha_pago,
                            monto_pago,
                            canal_pago,
                            referencia
                        )
                        VALUES
                        (
                            :1,
                            TRUNC(SYSDATE),
                            :2,
                            :3,
                            :4
                        )
                    """, [
                        int(fila["RESERVA_ID"]),
                        monto,
                        medio,
                        referencia
                    ])

                    # Si el saldo queda en 0,
                    # actualizo el estado de la reserva
                    if monto >= saldo:

                        cursor.execute("""
                            UPDATE ht_reservas
                            SET estado = 'PAGÓ'
                            WHERE reserva_id = :1
                        """, [int(fila["RESERVA_ID"])])

                    conn.commit()
                    conn.close()

                    st.success(
                        f"Pago registrado correctamente "
                        f"por ${monto:,.0f}"
                    )

                except Exception as e:


# hice consultas para mostrar
# habitaciones disponibles y reservas usando JOIN entre tablas. Después agregué
# formularios que funcionaran para registrar reservas y pagos directamente en la base
# de datos usando INSERT y commit(). También usé validaciones para evitar errores,
# calculé saldos pendientes con SQL usando SUM, NVL y HAVING, y actualicé el estado
# de la reserva cuando el pago completaba el saldo. La idea para el parcial fue que la aplicación
# no solo mostrara información, sino que realmente leyera y escribiera datos en Oracle.

                    st.error(f"Error: {e}")
